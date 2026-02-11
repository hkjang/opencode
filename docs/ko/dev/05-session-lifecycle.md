# 05. 세션 생명주기

이 문서는 OpenCode의 세션(Session) 생명주기를 설명한다. 세션 생성부터 메시지 처리, LLM 스트리밍, 도구 실행까지의 전체 흐름을 추적한다.

**소스 파일:**
- `packages/opencode/src/session/index.ts` — 세션 관리
- `packages/opencode/src/session/processor.ts` — 스트림 처리 루프
- `packages/opencode/src/session/message-v2.ts` — 메시지 구조
- `packages/opencode/src/session/llm.ts` — LLM 호출

## 전체 흐름 다이어그램

```
사용자 입력
    │
    ▼
1. Session.create() 또는 Session.get()
    │
    ▼
2. MessageV2 (사용자 메시지) 저장
    │
    ▼
3. Agent.get() → 모델, 도구, 권한 결정
    │
    ▼
4. MessageV2.toModelMessages() → AI SDK 형식 변환
    │
    ▼
5. LLM.stream() → AI SDK streamText() 호출
    │
    ▼
6. SessionProcessor.create().process()
    │
    ├── text-delta → 텍스트 파트 업데이트
    ├── tool-call → 도구 실행 (권한 확인 포함)
    ├── tool-result → 결과 기록
    ├── step-finish → 스냅샷 기록
    └── finish → 토큰/비용 계산
    │
    ▼
7. 반환: "continue" | "compact" | "stop"
    │
    ├── "continue" → 5번으로 (도구 결과를 바탕으로 재호출)
    ├── "compact" → 메시지 압축 후 5번으로
    └── "stop" → 세션 완료
```

## 1. 세션 생성

```typescript
// packages/opencode/src/session/index.ts
const session = await Session.create({
  // 선택적: parentID (포크 시)
})
```

세션은 다음 정보를 포함한다:

```typescript
export const Info = z.object({
  id: z.string(),
  projectID: z.string(),
  directory: z.string(),
  title: z.string(),        // "New session - 2025-01-15"
  slug: z.string(),
  parentID: z.string().optional(),
  permission: Ruleset.optional(),
  time: z.object({
    created: z.number(),
    updated: z.number(),
  }),
})
```

생성 후 `Session.Event.Created`가 발행된다.

## 2. 메시지 구조 (MessageV2)

### 사용자 메시지

```typescript
// MessageV2.User
{
  role: "user",
  time: { created: Date.now() },
  agent: "build",
  model: { providerID: "anthropic", modelID: "claude-sonnet-4-20250514" },
  summary: { title: "...", body: "...", diffs: [...] },
  system: "선택적 추가 지시사항",
  tools: { bash: true, "file.write": false },
  variant: "creative",
}
```

### 어시스턴트 메시지

```typescript
// MessageV2.Assistant
{
  role: "assistant",
  parentID: "user-message-id",
  agent: "build",
  providerID: "anthropic",
  modelID: "claude-sonnet-4-20250514",
  time: { created: Date.now(), completed: undefined },
  tokens: { input: 0, output: 0, reasoning: 0, cache: { read: 0, write: 0 } },
  cost: 0,
  error: undefined,
  finish: undefined,
}
```

### 메시지 파트 (Part)

메시지는 여러 파트로 구성된다:

| 타입 | 설명 |
|------|------|
| `TextPart` | 텍스트 응답 (타임스탬프 포함) |
| `ReasoningPart` | 추론 내용 (시작/종료 시간) |
| `ToolPart` | 도구 호출 (pending → running → completed/error) |
| `SnapshotPart` | 파일 시스템 스냅샷 |
| `PatchPart` | 파일 변경 diff |
| `StepStartPart` / `StepFinishPart` | 단계 경계 (타이밍) |
| `RetryPart` | 재시도 기록 |
| `SubtaskPart` | 하위 작업 |
| `AgentPart` | 에이전트 실행 마커 |
| `CompactionPart` | 압축 마커 |

## 3. AI SDK 형식 변환

`MessageV2.toModelMessages()`는 내부 메시지 형식을 AI SDK가 이해하는 형식으로 변환한다.

```typescript
export function toModelMessages(
  messages: MessageV2.WithParts[],
  model: Provider.Model
): ModelMessage[]
```

**프로바이더별 차이 처리:**
- **Anthropic/Claude/Bedrock**: 도구 결과에 미디어(이미지 등)를 직접 포함
- **OpenAI**: 미디어를 도구 결과에서 분리하여 별도 사용자 메시지로 주입
- **공통**: 완료/에러 상태의 도구 호출만 포함, 대기 중인 호출은 제외

## 4. LLM 스트리밍

```typescript
// packages/opencode/src/session/llm.ts
export function stream(input: StreamInput): StreamTextResult

interface StreamInput {
  user: MessageV2.User
  sessionID: string
  model: Provider.Model
  agent: Agent.Info
  system: string[]        // 시스템 프롬프트 배열
  abort: AbortSignal
  messages: ModelMessage[]
  tools: Record<string, Tool>
  retries?: number
}
```

**시스템 프롬프트 조합 순서:**
1. 에이전트 기본 프롬프트
2. 프로바이더별 추가 프롬프트
3. 사용자 커스텀 시스템 프롬프트
4. 메시지별 system 필드

**도구 필터링:**
- 에이전트 권한에 의해 거부된 도구 제외
- 사용자가 메시지에서 비활성화한 도구 제외
- LiteLLM 호환성: 이전 메시지에 도구 호출이 있지만 현재 활성 도구가 없으면 `_noop` 더미 도구 추가

**플러그인 훅:**
- `experimental.chat.system.transform` — 시스템 프롬프트 변환
- `chat.params` — temperature, topP 등 파라미터
- `chat.headers` — 커스텀 HTTP 헤더

## 5. 프로세서 (SessionProcessor)

프로세서는 LLM 스트림의 이벤트를 처리하는 핵심 루프이다.

```typescript
// packages/opencode/src/session/processor.ts
const processor = SessionProcessor.create({
  sessionID,
  agent,
  model,
  abort,
  // ...
})

const result = await processor.process(stream)
```

### 스트림 이벤트 처리

| 이벤트 | 처리 |
|--------|------|
| `start` | 초기 설정 |
| `text-start` | TextPart 생성 |
| `text-delta` | TextPart에 텍스트 누적 |
| `text-end` | TextPart 완료 타임스탬프 기록 |
| `reasoning-start` | ReasoningPart 생성 |
| `reasoning-delta` | 추론 텍스트 누적 |
| `reasoning-end` | ReasoningPart 완료 |
| `tool-input` | 도구 입력 스트리밍 (UI 미리보기용) |
| `tool-call` | 도구 호출 실행, ToolPart 상태 업데이트 |
| `tool-result` | 도구 결과 기록 |
| `tool-call-error` | 도구 에러 기록 |
| `step-start` | 스냅샷 기록 시작 |
| `step-finish` | 스냅샷 완료, PatchPart 생성 |
| `finish` | 토큰 사용량, 비용, 완료 사유 기록 |
| `error` | 에러 변환 및 기록 |

### 반환값

- **`"continue"`**: 도구 실행 후 LLM에 결과를 반환해야 함 → 다시 스트리밍
- **`"compact"`**: 컨텍스트 초과 → 메시지 압축 후 재시도
- **`"stop"`**: 대화 완료 또는 에러

### 둠 루프(Doom loop) 감지

프로세서는 마지막 3회의 도구 호출이 동일한 입력으로 반복되는지 추적한다. 반복이 감지되면 에이전트에 알림을 전달하여 무한 루프를 방지한다.

### 권한 거부 처리

도구 실행 중 `PermissionNext.RejectedError`나 `CorrectedError`가 발생하면:
- `blocked = true`로 설정
- 프로세서가 `"stop"`을 반환
- UI에 권한 거부 상태 표시

## 6. 세션 포크(Fork)

기존 세션에서 새 세션을 분기할 수 있다:

```typescript
const forked = await Session.fork({
  sessionID: "original-session-id",
})
```

포크는 원본 세션의 모든 메시지를 새 ID로 복사한다.

## 7. 비용 계산

```typescript
export function getUsage(sessionID: string): {
  totalTokens: number
  totalCost: number
  // 프로바이더별 상세
}
```

프로바이더별로 토큰 비용 계산이 다르다:
- **캐시 토큰**: 일부 프로바이더는 캐시 읽기/쓰기 토큰에 별도 요금 적용
- **추론 토큰**: 출력 토큰 요금으로 계산
- **OpenRouter**: 별도의 비용 포맷 사용

## 참고 문서

- [01. 아키텍처 개요](./01-architecture-overview.md) — 요청 처리 흐름 요약
- [04. 에이전트 시스템](./04-agent-system.md) — 에이전트 선택
- [06. 프로바이더 시스템](./06-provider-system.md) — LLM SDK 호출
- [07. 도구 시스템](./07-tool-system.md) — 도구 실행
- [08. 이벤트 버스](./08-bus-event-system.md) — 세션 이벤트
