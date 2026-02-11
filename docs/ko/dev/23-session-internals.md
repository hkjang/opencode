# 23. 세션 내부 구현

이 문서는 세션 처리의 내부 구현을 상세히 설명한다. 프롬프트 오케스트레이션, 컨텍스트 압축, 재시도, 상태 추적, 지시사항 로딩, 되돌리기를 다룬다.

**소스 파일:**
- `packages/opencode/src/session/prompt.ts` — 프롬프트 오케스트레이션 (65KB)
- `packages/opencode/src/session/compaction.ts` — 컨텍스트 압축
- `packages/opencode/src/session/retry.ts` — 재시도 로직
- `packages/opencode/src/session/status.ts` — 세션 상태 추적
- `packages/opencode/src/session/instruction.ts` — 지시사항 로딩
- `packages/opencode/src/session/revert.ts` — 되돌리기

## SessionPrompt: 오케스트레이션 허브

`SessionPrompt`는 세션 처리의 핵심 오케스트레이터이다. 사용자 메시지 생성, LLM 호출 루프, 도구 해석, 하위 작업 처리를 모두 조율한다.

### 주요 함수

```typescript
// packages/opencode/src/session/prompt.ts
export namespace SessionPrompt {
  // 사용자 메시지 생성 + 처리 루프 시작
  export async function prompt(input: PromptInput): Promise<void>

  // 핵심 이벤트 루프 — LLM 스트리밍 + 도구 실행
  export async function loop(input: LoopInput): Promise<void>

  // 셸 명령 실행
  export async function shell(input: ShellInput): Promise<void>

  // 명령어 기반 작업 실행
  export async function command(input: CommandInput): Promise<void>

  // 진행 중인 처리 취소
  export async function cancel(sessionID: string): Promise<void>

  // 템플릿 문자열에서 파일 참조 등을 해석
  export function resolvePromptParts(template: string): string
}
```

### prompt() 흐름

```
prompt(input)
  │
  ├── 1. 사용자 메시지 생성 (MessageV2.User)
  │     ├── 에이전트 결정 (Agent.get)
  │     ├── 모델 결정 (Provider.getModel)
  │     └── 시스템 프롬프트 조합
  │
  ├── 2. Storage에 메시지 저장
  │
  ├── 3. 어시스턴트 메시지 생성 (MessageV2.Assistant)
  │
  └── 4. loop() 호출 → LLM 스트리밍 시작
```

### loop() 핵심 루프

```
loop(input)
  │
  ├── SessionStatus.set("busy")
  │
  ├── 메시지 히스토리 로드 (MessageV2.stream → filterCompacted)
  │
  ├── AI SDK 형식 변환 (MessageV2.toModelMessages)
  │
  ├── 도구 해석 (ToolRegistry.tools + MCP.tools)
  │
  ├── LLM.stream() 호출
  │
  ├── SessionProcessor.process() → 스트림 이벤트 처리
  │     │
  │     ├── "continue" → 도구 결과 반영 후 루프 재시작
  │     ├── "compact"  → 압축 처리 후 루프 재시작
  │     └── "stop"     → 루프 종료
  │
  ├── 에러 처리
  │     ├── APIError + retryable → retry 대기 후 재시도
  │     ├── ContextOverflow → 압축 시도
  │     └── 기타 에러 → 세션 에러 기록
  │
  └── SessionStatus.set("idle")
```

### 플러그인 훅 통합

`prompt.ts`에서 호출하는 플러그인 훅:

| 훅 | 시점 | 설명 |
|----|------|------|
| `tool.execute.before` | 도구 실행 전 | 도구 입력 가로채기 |
| `tool.execute.after` | 도구 실행 후 | 도구 결과 가로채기 |
| `chat.message` | 메시지 생성 시 | 메시지 변환 |
| `experimental.chat.messages.transform` | LLM 호출 전 | 메시지 목록 변환 |
| `experimental.session.compacting` | 압축 전 | 커스텀 압축 로직 |
| `shell.env` | 셸 실행 시 | 환경 변수 주입 |

## SessionCompaction: 컨텍스트 압축

### 개요

컨텍스트 창(context window)이 한계에 가까워지면 이전 대화를 요약하여 토큰을 절약한다.

### 주요 함수

```typescript
// packages/opencode/src/session/compaction.ts
export namespace SessionCompaction {
  // 토큰 사용량이 컨텍스트 한계를 초과했는지 확인
  export function isOverflow(tokens: number, model: Provider.Model): boolean

  // 도구 출력을 선택적으로 제거하여 토큰 절약
  export function prune(messages: MessageV2.WithParts[]): MessageV2.WithParts[]

  // AI로 대화 요약 생성
  export async function process(input: {
    sessionID: string
    messages: MessageV2.WithParts[]
    model: Provider.Model
    agent: Agent.Info
    abort: AbortSignal
  }): Promise<void>

  // 압축 요청 대기열에 추가
  export function create(sessionID: string): void
}
```

### 압축 과정

```
1. isOverflow() 감지
   └── tokens > model.limit.context - config.compaction.reserved

2. prune() — 1차 절약
   └── 오래된 도구 출력 제거 (skill 도구는 보존)
   └── 각 도구 결과를 "[pruned]"로 교체

3. process() — 2차 절약 (여전히 초과 시)
   └── "compaction" 에이전트로 요약 생성
   └── CompactionPart 추가 (요약 결과)
   └── 이후 loop()에서 CompactionPart 이전 메시지는 무시
```

### 설정

```jsonc
// opencode.json
{
  "compaction": {
    "auto": true,       // 자동 압축 활성화 (기본: true)
    "reserved": 20000,  // 예약 토큰 수
    "prune": true       // 도구 출력 사전 제거 (기본: true)
  }
}
```

### 이벤트

```typescript
export const Event = {
  Compacted: BusEvent.define("session.compacted", z.object({
    sessionID: z.string(),
  })),
}
```

## SessionRetry: 재시도 로직

### 개요

LLM API 호출 실패 시 지수 백오프(exponential backoff)로 재시도한다.

```typescript
// packages/opencode/src/session/retry.ts
export namespace SessionRetry {
  // 취소 가능한 대기
  export function sleep(ms: number, signal: AbortSignal): Promise<void>

  // 백오프 지연 시간 계산
  export function delay(attempt: number, error?: Error): number

  // 재시도 가능한 에러인지 판단
  export function retryable(error: Error): boolean
}
```

### 백오프 전략

```
지연 시간 = INITIAL_DELAY × BACKOFF_FACTOR^attempt

INITIAL_DELAY = 2000ms
BACKOFF_FACTOR = 2

attempt 0: 2초
attempt 1: 4초
attempt 2: 8초
attempt 3: 16초
...

최대 지연: 30초 (Retry-After 헤더 없는 경우)
```

### HTTP 헤더 지원

서버가 `Retry-After` 또는 `Retry-After-Ms` 헤더를 반환하면 해당 값을 우선 사용한다.

### 재시도 가능한 에러

- Rate limit 에러 (429)
- 서버 과부하 (503, 529)
- 연결 초기화
- 프리 사용량 한도 초과
- **재시도 불가:** `ContextOverflowError`, 인증 에러

## SessionStatus: 상태 추적

### 상태 타입

```typescript
// packages/opencode/src/session/status.ts
export const Info = z.discriminatedUnion("type", [
  z.object({ type: z.literal("idle") }),
  z.object({
    type: z.literal("retry"),
    attempt: z.number(),
    message: z.string(),
    next: z.number(),     // 다음 재시도 시각 (timestamp)
  }),
  z.object({ type: z.literal("busy") }),
])
```

### API

```typescript
export namespace SessionStatus {
  export function get(sessionID: string): Info
  export function list(): Record<string, Info>
  export function set(sessionID: string, info: Info): void
}
```

`set()` 호출 시 `session.status` 이벤트가 발행되어 UI가 상태 변화를 반영한다.

## InstructionPrompt: 지시사항 로딩

### 개요

프로젝트, 사용자, 전역 수준의 지시사항 파일을 로드하여 시스템 프롬프트에 포함한다.

### 지시사항 파일 검색

```
프로젝트 레벨:
  ├── AGENTS.md
  ├── CLAUDE.md
  └── CONTEXT.md

전역 레벨:
  ├── ~/.claude/CLAUDE.md
  └── ~/.config/opencode/AGENTS.md

원격:
  └── config.instructions[] 내 HTTP/HTTPS URL
```

### 주요 함수

```typescript
// packages/opencode/src/session/instruction.ts
export namespace InstructionPrompt {
  // 모든 지시사항을 시스템 프롬프트 문자열로 반환
  export function system(): Promise<string[]>

  // 지시사항 파일 경로 목록
  export function systemPaths(): Promise<string[]>

  // 디렉터리별 지시사항 검색
  export function resolve(directory: string): Promise<string[]>

  // 특정 디렉터리에서 지시사항 파일 찾기 (findUp 탐색)
  export function find(directory: string): Promise<string | null>
}
```

### findUp 탐색

`find()`는 현재 디렉터리에서 시작하여 상위 디렉터리를 탐색하며 지시사항 파일을 찾는다. 프로젝트 경계(Git 루트)를 넘지 않는다.

## SessionRevert: 되돌리기

### 개요

사용자가 특정 메시지 시점으로 대화를 되돌릴 수 있다. 파일 시스템 변경도 함께 복원된다.

### 주요 함수

```typescript
// packages/opencode/src/session/revert.ts
export namespace SessionRevert {
  // 특정 메시지/파트 시점으로 되돌리기
  export async function revert(input: {
    messageID: string
    partID?: string
    snapshot?: Snapshot   // 복원할 파일 상태
    diff?: FileDiff[]     // 되돌릴 파일 변경
  }): Promise<void>

  // 되돌린 상태를 다시 복원
  export async function unrevert(): Promise<void>

  // 되돌리기 후 후속 메시지 삭제
  export async function cleanup(sessionID: string): Promise<void>
}
```

### 되돌리기 과정

```
1. revert()
   ├── Snapshot.revert(patches) — 파일을 이전 상태로 복원
   ├── 대상 메시지 이후의 모든 메시지 삭제
   └── Session.Event.Diff 이벤트 발행

2. unrevert()
   ├── Snapshot.restore(snapshot) — 되돌리기 전 상태로 복원
   └── Session.Event.Diff 이벤트 발행
```

## 참고 문서

- [05. 세션 생명주기](./05-session-lifecycle.md) — 세션 처리 흐름 개요
- [06. 프로바이더 시스템](./06-provider-system.md) — LLM 호출
- [07. 도구 시스템](./07-tool-system.md) — 도구 실행
- [08. 이벤트 버스](./08-bus-event-system.md) — 세션 이벤트
- [14. 저장소와 상태](./14-storage-and-state.md) — 스냅샷 시스템
