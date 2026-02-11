# 01. 아키텍처 개요

이 문서는 OpenCode의 전체 시스템 아키텍처, 요청 흐름, 계층 구조를 설명한다.

## 시스템 개요

OpenCode는 TypeScript 기반의 AI 코딩 에이전트(Agent)로, 다수의 LLM 프로바이더(Provider)를 통합하여 터미널(TUI), 웹, 데스크톱 인터페이스를 제공한다. 핵심 설계 원칙은 다음과 같다:

- **모듈 격리**: 네임스페이스(Namespace) 패턴으로 각 모듈의 경계를 명확히 함
- **타입 안전성**: Zod 스키마로 런타임 검증과 TypeScript 타입을 동시에 보장
- **이벤트 기반**: 이벤트 버스(Event Bus)로 컴포넌트 간 느슨한 결합
- **확장성**: 플러그인(Plugin)과 MCP 프로토콜로 외부 기능 통합

## 계층 구조

```
┌─────────────────────────────────────────────────────┐
│                    클라이언트 계층                      │
│          TUI (SolidJS)  │  Web  │  Desktop (Tauri)  │
└────────────────────┬────────────────────────────────┘
                     │ HTTP / SSE
┌────────────────────▼────────────────────────────────┐
│                    서버 계층 (Hono)                    │
│    라우트 · 미들웨어 · SSE · OpenAPI · CORS · 인증    │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│                    세션 계층                           │
│   Session · Processor · MessageV2 · LLM 스트리밍     │
└──────┬───────────────────────────────┬──────────────┘
       │                               │
┌──────▼──────┐                 ┌──────▼──────┐
│  에이전트    │                 │  프로바이더   │
│ build/plan/ │                 │ 20+ LLM     │
│ general/... │                 │ SDK 통합     │
└──────┬──────┘                 └─────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│                    도구 계층                           │
│  40+ 내장 도구 · ToolRegistry · MCP 도구 · 플러그인   │
└─────────────────────────────────────────────────────┘
       │
┌──────▼──────────────────────────────────────────────┐
│                    인프라 계층                         │
│  Bus · Config · Storage · Permission · Instance      │
└─────────────────────────────────────────────────────┘
```

## 요청 처리 흐름

사용자의 메시지가 처리되는 전체 흐름을 추적한다.

### 1. 클라이언트 → 서버

사용자 입력은 클라이언트(TUI/Web/Desktop)에서 HTTP API를 통해 서버로 전달된다.

```
사용자 입력 → POST /session/:id/message → 서버 라우트
```

### 2. 세션 생성 및 메시지 기록

```typescript
// packages/opencode/src/session/index.ts
const session = await Session.create({ ... })
```

세션(Session)이 생성되고, 사용자 메시지가 `MessageV2` 형식으로 저장소(Storage)에 기록된다.

### 3. 에이전트 및 모델 결정

```typescript
// packages/opencode/src/agent/agent.ts
const agent = Agent.get("build") // 기본 에이전트
```

에이전트 설정에 따라 사용할 모델, 시스템 프롬프트, 허용 도구가 결정된다.

### 4. LLM 스트리밍

```typescript
// packages/opencode/src/session/llm.ts
const result = LLM.stream({
  user,
  sessionID,
  model,
  agent,
  system,
  messages,
  tools,
  abort,
})
```

AI SDK의 `streamText()`를 통해 LLM 응답이 스트리밍된다.

### 5. 프로세서의 스트림 처리

```typescript
// packages/opencode/src/session/processor.ts
const processor = SessionProcessor.create({ ... })
const result = await processor.process(stream)
// 반환값: "continue" | "compact" | "stop"
```

프로세서(Processor)는 스트림 이벤트를 처리하며, 각 이벤트 타입에 따라 메시지 파트(Part)를 업데이트한다:

- `text-delta`: 텍스트 응답 누적
- `tool-call`: 도구 호출 실행
- `tool-result`: 도구 결과 기록
- `step-finish`: 단계 완료 및 스냅샷(Snapshot) 기록
- `finish`: 최종 토큰 사용량 및 비용 계산

### 6. 도구 실행

LLM이 도구 호출을 요청하면:

1. `ToolRegistry`에서 해당 도구를 찾음
2. `PermissionNext`로 권한 확인 (allow/deny/ask)
3. 도구 실행 후 결과를 LLM에 반환
4. LLM이 결과를 바탕으로 다음 응답 생성

### 7. 이벤트 발행 및 UI 업데이트

모든 상태 변경은 이벤트 버스(Bus)를 통해 발행된다:

```typescript
// 세션 이벤트
Bus.publish(Session.Event.Updated, { ... })

// 메시지 파트 업데이트
Bus.publish(MessageV2.Event.PartUpdated, { ... })
```

서버의 SSE 엔드포인트(`GET /event`)가 이벤트를 클라이언트에 실시간으로 전달한다.

## 핵심 모듈 관계

```
Config ──────────────────────────────────────────────┐
  │                                                  │
  ├── Provider (LLM SDK 인스턴스 생성)                │
  │     └── Agent (에이전트별 모델·권한·프롬프트)       │
  │           └── Session (대화 단위 관리)             │
  │                 └── Processor (스트림 처리 루프)    │
  │                       ├── LLM (AI SDK 호출)       │
  │                       ├── ToolRegistry (도구 조회) │
  │                       └── Snapshot (파일 상태 기록)│
  │                                                  │
  ├── Permission (권한 규칙 평가)                      │
  ├── Plugin (확장 기능 로드)                          │
  ├── MCP (외부 도구 서버 연동)                        │
  └── Storage (파일 기반 영속 저장)                     │
                                                     │
Bus ◀──── 모든 모듈이 이벤트를 발행 ────────────────────┘
  │
  └── Server (SSE) → 클라이언트
```

## Instance: 프로젝트 컨텍스트

`Instance`는 OpenCode의 프로젝트 단위 컨텍스트를 관리하는 핵심 모듈이다.

```typescript
// packages/opencode/src/project/instance.ts
await Instance.provide({ directory: "/path/to/project" }, async () => {
  // 이 콜백 내에서 Instance.directory, Instance.worktree 사용 가능
  const config = Config.get()
  const session = await Session.create({ ... })
})
```

- `Instance.provide()`: 디렉터리별 비동기 컨텍스트 생성 또는 재사용
- `Instance.state()`: 디렉터리에 종속된 지연 초기화(Lazy initialization) 상태
- `Instance.dispose()`: 인스턴스 정리 (모든 state 해제)

모든 핵심 모듈(`Config`, `Session`, `Storage`, `Bus` 등)은 `Instance.state()`를 통해 프로젝트별로 격리된 상태를 관리한다.

## 참고 문서

- [02. 모노레포 구조](./02-monorepo-structure.md) — 패키지 구성 상세
- [03. 핵심 패턴](./03-core-patterns.md) — Namespace, Zod, Instance.state() 패턴 상세
- [05. 세션 생명주기](./05-session-lifecycle.md) — 요청 흐름의 상세 설명
- [08. 이벤트 버스](./08-bus-event-system.md) — Bus 시스템 상세
