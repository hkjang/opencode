# 30. 질문과 사용자 상호작용

이 문서는 OpenCode의 사용자 상호작용 패턴을 설명한다. 질문(Question) 시스템, 권한 요청 다이얼로그, 사용자 피드백 루프를 다룬다.

**소스 파일:**
- `packages/opencode/src/question/index.ts` — 질문 시스템
- `packages/opencode/src/permission/next.ts` — 권한 요청
- `packages/opencode/src/session/status.ts` — 세션 상태

## 질문 시스템 (Question)

### 개요

질문 시스템은 에이전트가 작업 중 사용자에게 선택지를 제시하거나 정보를 요청할 때 사용한다. Promise 기반으로 사용자 응답을 대기하며, Bus 이벤트로 UI에 다이얼로그를 표시한다.

### Question.Info 스키마

```typescript
// packages/opencode/src/question/index.ts
export const Info = z.object({
  question: z.string(),               // 질문 텍스트
  header: z.string().max(30),          // 짧은 헤더 (UI 라벨)
  options: z.array(z.object({
    label: z.string(),                 // 옵션 표시 텍스트
    description: z.string(),           // 옵션 설명
  })),
  multiple: z.boolean().optional(),    // 복수 선택 허용
  custom: z.boolean().optional(),      // 자유 텍스트 입력 허용
})
```

### 주요 함수

```typescript
export namespace Question {
  // 사용자에게 질문 (UI 다이얼로그 표시)
  export async function ask(input: {
    sessionID: string
    questions: Info[]
    tool?: { messageID: string, callID: string }
  }): Promise<Answer[]>

  // 대기 중인 질문에 응답
  export async function reply(input: {
    id: string
    answers: Answer[]
  }): Promise<void>

  // 질문 거부
  export async function reject(input: {
    id: string
    message?: string
  }): Promise<void>

  // 대기 중인 질문 목록
  export function list(): Request[]

  // 질문 거부 에러
  export const RejectedError: NamedError
}
```

### 질문 흐름

```
에이전트 (Question 도구 호출)
    │
    ├── 1. Question.ask() 호출
    │     └── 대기 중 요청 맵에 등록
    │
    ├── 2. Event.Asked 이벤트 발행
    │     └── UI가 다이얼로그 렌더링
    │
    ├── 3. 사용자 응답 대기 (Promise)
    │
    ├── 4a. Question.reply() → 답변 반환
    │     └── Event.Replied 이벤트 발행
    │
    └── 4b. Question.reject() → RejectedError 발생
          └── Event.Rejected 이벤트 발행
          └── 에이전트에 거부 사유 전달
```

### 이벤트

```typescript
export const Event = {
  Asked: BusEvent.define("question.asked", z.object({
    request: Request,
  })),
  Replied: BusEvent.define("question.replied", z.object({
    id: z.string(),
    answers: z.array(z.array(z.string())),
  })),
  Rejected: BusEvent.define("question.rejected", z.object({
    id: z.string(),
  })),
}
```

## 권한 요청 상호작용

### 개요

권한 시스템도 질문과 유사한 사용자 상호작용 패턴을 사용하지만, 더 구조화된 allow/deny/ask 규칙을 가진다.

### 권한 요청 흐름

```
도구 실행 시 ctx.ask() 호출
    │
    ├── 룰셋 평가
    │     ├── "allow" → 즉시 허용 (UI 표시 없음)
    │     ├── "deny"  → DeniedError (UI 표시 없음)
    │     └── "ask"   → 사용자에게 물어봄
    │
    ├── Event.Asked 이벤트 발행
    │     └── UI에 권한 요청 다이얼로그 표시
    │         ├── 허용 내용: 도구 이름, 패턴, 메타데이터
    │         └── 선택지: "이번만 허용" / "항상 허용" / "거부"
    │
    └── 사용자 선택
          ├── "once"   → 이번 요청만 허용
          ├── "always" → 룰셋에 영구 추가 + 같은 세션의 유사 요청도 승인
          └── "reject" → RejectedError 또는 CorrectedError (피드백 포함)
```

### 권한 다이얼로그 UI 데이터

```typescript
// UI에 전달되는 권한 요청 데이터
const request = {
  id: "per_018f9a...",
  sessionID: "ses_018f9a...",
  permission: "bash",            // 권한 유형
  patterns: ["npm test"],        // 요청 패턴
  metadata: {
    command: "npm test",         // 사람이 읽을 수 있는 정보
    description: "Run tests",
  },
  tool: "bash",                  // 도구 이름
}
```

## 상호작용 패턴 비교

| 특성 | Question | Permission |
|------|----------|------------|
| 호출자 | 에이전트 (Question 도구) | 도구 (ctx.ask) |
| 응답 타입 | 자유 텍스트 / 선택지 | once / always / reject |
| 규칙 기반 자동화 | 불가 | 가능 (룰셋) |
| 영구 저장 | 불가 | "always" → Storage에 저장 |
| 거부 시 동작 | RejectedError | RejectedError / CorrectedError |
| 세션 간 전파 | 불가 | "always" 규칙은 세션 간 유지 |

## 세션 상태와 UI 연동

### 상태 타입

```typescript
// packages/opencode/src/session/status.ts
type Info =
  | { type: "idle" }
  | { type: "busy" }
  | { type: "retry", attempt: number, message: string, next: number }
```

### UI 상태 표시

| 상태 | UI 표시 |
|------|---------|
| `idle` | 입력 대기 |
| `busy` | 스피너 + "처리 중..." |
| `retry` | "재시도 대기 (N회째, M초 후)" |

### 상태 전환

```
사용자 입력 → busy
    │
    ├── 정상 완료 → idle
    ├── 권한 요청 → busy (대기)
    ├── 질문 → busy (대기)
    ├── API 에러 → retry → (재시도) → busy
    └── 취소 → idle
```

## 프론트엔드 구현 패턴

### TUI (터미널)

TUI에서는 SolidJS 컨텍스트를 사용하여 상호작용 상태를 관리한다:

```
SDKProvider
  └── SyncProvider        ← SSE로 이벤트 수신
       └── LocalProvider  ← 로컬 상태 관리
            └── 다이얼로그
                 ├── dialog-agent   ← 에이전트 선택
                 ├── dialog-model   ← 모델 선택
                 ├── dialog-mcp     ← MCP 관리
                 └── (권한/질문 다이얼로그)
```

### Web UI

웹에서는 별도의 컨텍스트 프로바이더로 권한과 질문을 관리한다:

```typescript
// packages/app/src/context/permission.tsx
// SSE 이벤트를 구독하여 권한 요청 다이얼로그 표시

// packages/app/src/components/question-dock.tsx
// 질문 다이얼로그 UI 구현
```

## 참고 문서

- [09. 권한 시스템](./09-permission-system.md) — 권한 규칙 상세
- [08. 이벤트 버스](./08-bus-event-system.md) — 이벤트 흐름
- [16. 프론트엔드 아키텍처](./16-frontend-architecture.md) — TUI/Web UI 구조
- [25. 도구 구현 상세](./25-tool-implementation-details.md) — Question 도구
