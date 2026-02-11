# 28. Agent Client Protocol (ACP)

이 문서는 OpenCode의 ACP(Agent Client Protocol) 구현을 설명한다. 외부 IDE나 도구가 OpenCode 에이전트와 표준화된 방식으로 상호작용할 수 있게 한다.

**소스 파일:**
- `packages/opencode/src/acp/agent.ts` — ACP 프로토콜 에이전트 (57KB)
- `packages/opencode/src/acp/session.ts` — 세션 상태 관리
- `packages/opencode/src/acp/types.ts` — 타입 정의

## 개요

ACP는 AI 에이전트와 클라이언트(IDE, 편집기 등) 간의 표준 통신 프로토콜이다. OpenCode는 `@agentclientprotocol/sdk`를 사용하여 ACP 서버를 구현한다. 이를 통해 외부 클라이언트가 세션 생성, 메시지 전송, 권한 응답, 도구 상태 확인 등을 수행할 수 있다.

## 아키텍처

```
외부 IDE / 클라이언트
    │
    │  ACP 프로토콜 (JSON-RPC)
    │
    ▼
ACP.Agent (agent.ts)
    │
    ├── ACPSessionManager (session.ts)
    │     └── OpenCode SDK 클라이언트
    │
    └── OpenCode 서버
          ├── Session
          ├── Permission
          ├── Provider
          └── Agent
```

## ACP.Agent 클래스

```typescript
// packages/opencode/src/acp/agent.ts
export class Agent implements AgentSideConnection {
  constructor(
    connection: AgentSideConnection,
    config: ACPConfig
  )

  // ACP 프로토콜 구현
  async initialize(capabilities: ClientCapabilities): Promise<ServerCapabilities>
  async createSession(params: CreateSessionParams): Promise<SessionInfo>
  async loadSession(params: LoadSessionParams): Promise<SessionInfo>
  async prompt(params: PromptParams): Promise<PromptResult>
  // ...
}
```

### 초기화

```typescript
interface ACPConfig {
  sdk: OpencodeClient    // OpenCode SDK 클라이언트
  defaultModel?: string  // 기본 모델 ("providerID/modelID")
}
```

ACP 에이전트는 초기화 시 다음 기능(Capability)을 광고한다:
- 세션 생성/로드
- 메시지 전송
- 파일 읽기/쓰기
- MCP 서버 지원

## ACPSessionManager

```typescript
// packages/opencode/src/acp/session.ts
export class ACPSessionManager {
  // 새 세션 생성
  async create(
    cwd: string,
    mcpServers: McpServer[],
    model?: string
  ): Promise<ACPSessionState>

  // 기존 세션 로드
  async load(
    sessionId: string,
    cwd: string,
    mcpServers: McpServer[],
    model?: string
  ): Promise<ACPSessionState>

  // 모델/변형 관리
  getModel(): string | undefined
  setModel(model: string): void
  getVariant(): string | undefined
  setVariant(variant: string): void
  setMode(modeId: string): void
}
```

### 세션 상태

```typescript
interface ACPSessionState {
  id: string
  cwd: string
  mcpServers: McpServer[]
  createdAt: Date
  model?: string
  variant?: string
  modeId?: string
}
```

## 이벤트 구독

ACP 에이전트는 OpenCode의 Bus 이벤트를 구독하여 ACP 프로토콜 이벤트로 변환한다:

```
OpenCode Bus 이벤트                    ACP 프로토콜 이벤트
─────────────────                      ──────────────────
PermissionNext.Event.Asked      →      permission.request
MessageV2.Event.PartUpdated     →      tool.state.update
Session.Event.Updated           →      session.updated
```

### 권한 처리

```
1. OpenCode가 도구 실행 시 권한 요청 (PermissionNext.ask)
2. ACP 에이전트가 Event.Asked 수신
3. ACP 프로토콜로 클라이언트에 권한 요청 전달
4. 클라이언트에서 사용자가 응답 (once/always/reject)
5. ACP 에이전트가 PermissionNext.reply() 호출
```

### 도구 상태 보고

도구 실행 상태가 변경될 때마다 ACP 클라이언트에 알린다:

```
pending  → 도구 호출 대기
running  → 도구 실행 중
completed → 도구 실행 완료
error    → 도구 실행 실패
```

### Edit 권한의 특수 처리

파일 편집 권한의 경우, ACP 에이전트는 diff를 `applyPatch()` 형식으로 변환하여 클라이언트가 미리보기를 제공할 수 있게 한다.

## 사용량 추적

```typescript
// ACP를 통한 토큰 사용량 계산
const usage = {
  inputTokens: tokens.input,
  outputTokens: tokens.output,
  contextLimit: model.limit.context,
}
```

## 프로토콜 상태

| 기능 | 상태 |
|------|------|
| 초기화 + 기능 광고 | 구현 완료 |
| 세션 생성/로드 | 구현 완료 |
| 메시지 전송 | 구현 완료 |
| 파일 읽기/쓰기 | 구현 완료 |
| 권한 요청/응답 | 구현 완료 |
| MCP 서버 지원 | 구현 완료 |
| 스트리밍 응답 | 계획 중 |
| 도구 호출 상세 보고 | 계획 중 |
| 세션 영속성 | 계획 중 |

## 참고 문서

- [04. 에이전트 시스템](./04-agent-system.md) — 에이전트 설정
- [09. 권한 시스템](./09-permission-system.md) — 권한 처리
- [11. 서버 API](./11-server-api.md) — HTTP API (ACP와 병행)
