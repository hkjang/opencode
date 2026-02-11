# 08. 이벤트 버스 시스템

이 문서는 OpenCode의 이벤트 버스(Event Bus) 시스템을 설명한다. 모듈 간 느슨한 결합을 구현하는 발행-구독(Pub/Sub) 패턴의 핵심이다.

## 개요

이벤트 버스는 두 계층으로 구성된다:

- **Bus**: 인스턴스(Instance) 범위의 로컬 이벤트 버스
- **GlobalBus**: 프로세스 간 통신을 위한 전역 버스 (SSE 등)

모든 상태 변경(세션 생성, 메시지 업데이트, 권한 요청 등)은 Bus를 통해 발행되며, 클라이언트는 SSE를 통해 실시간으로 이벤트를 수신한다.

## BusEvent.define()

이벤트 타입은 `BusEvent.define()`으로 정의한다. 전역 레지스트리(Registry)에 자동으로 등록되어 OpenAPI 스펙 생성에도 활용된다.

```typescript
// packages/opencode/src/bus/bus-event.ts
export namespace BusEvent {
  export function define<T extends z.ZodObject<any>>(
    type: string,    // 이벤트 타입 문자열 (예: "session.updated")
    schema: T        // 이벤트 데이터의 Zod 스키마
  ): Definition<T>

  // 등록된 모든 이벤트의 판별 유니온 스키마 생성
  export function payloads(): z.ZodDiscriminatedUnion<...>
}
```

**내부 동작:**
1. `define()` 호출 시 전역 `Map<string, Definition>`에 등록
2. `payloads()`는 등록된 모든 이벤트를 `type` 필드 기준 판별 유니온(Discriminated union)으로 합침
3. 이 유니온은 서버의 SSE 엔드포인트와 OpenAPI 스펙에서 사용

## 이벤트 선언

각 모듈은 자체 네임스페이스 안에서 관련 이벤트를 선언한다.

```typescript
// packages/opencode/src/session/index.ts
export namespace Session {
  export const Event = {
    Created: BusEvent.define("session.created",
      z.object({ info: Info })
    ),
    Updated: BusEvent.define("session.updated",
      z.object({ info: Info })
    ),
    Deleted: BusEvent.define("session.deleted",
      z.object({ info: Info })
    ),
    Diff: BusEvent.define("session.diff",
      z.object({ /* diff 데이터 */ })
    ),
    Error: BusEvent.define("session.error",
      z.object({ sessionID: z.string(), error: z.any() })
    ),
  }
}
```

**주요 이벤트 모듈:**
- `Session.Event` — 세션 CRUD, diff, 에러
- `MessageV2.Event` — 메시지/파트 업데이트, 제거
- `PermissionNext.Event` — 권한 요청(Asked), 응답(Replied)
- `MCP.ToolsChanged` — MCP 서버 도구 목록 변경
- `Bus.InstanceDisposed` — 인스턴스 정리 시 발행

## Bus API

```typescript
// packages/opencode/src/bus/index.ts
export namespace Bus {
  // 이벤트 발행 (타입 안전)
  export function publish<D extends BusEvent.Definition>(
    event: D,
    properties: z.infer<D["properties"]>
  ): Promise<void>

  // 특정 이벤트 구독
  export function subscribe<D extends BusEvent.Definition>(
    event: D,
    callback: (properties: z.infer<D["properties"]>) => void
  ): () => void  // 구독 해제 함수

  // 모든 이벤트 구독
  export function subscribeAll(
    callback: (event: { type: string, properties: unknown }) => void
  ): () => void

  // 일회성 구독 (첫 이벤트 수신 후 자동 해제)
  export function once<D extends BusEvent.Definition>(
    event: D,
    callback: (properties: z.infer<D["properties"]>) => void
  ): () => void

  // 인스턴스 정리 이벤트
  export const InstanceDisposed: BusEvent.Definition
}
```

### 발행 흐름

```
Bus.publish(event, props)
  ├── 로컬 구독자에게 전달 (Instance 범위)
  │    └── subscriptions[event.type].forEach(cb => cb(props))
  ├── 와일드카드 구독자에게 전달
  │    └── subscriptions["*"].forEach(cb => cb({ type, props }))
  └── GlobalBus에 전달 (프로세스 간 통신)
       └── SSE 엔드포인트로 클라이언트에 스트리밍
```

### 인스턴스 생명주기

Bus는 `Instance.state()`를 사용하여 인스턴스별 구독 맵을 관리한다.

```typescript
// 내부 구조
const state = Instance.state(async () => ({
  subscriptions: new Map<string, Subscription[]>()
}), async (state) => {
  // 인스턴스 정리 시 InstanceDisposed 이벤트 발행
  Bus.publish(Bus.InstanceDisposed, {})
})
```

`Instance.dispose()` 호출 시:
1. `InstanceDisposed` 이벤트가 발행되어 정리 로직 실행
2. 모든 구독이 해제됨
3. 상태가 캐시에서 제거됨

## SSE 연동

서버의 SSE 엔드포인트는 Bus 이벤트를 실시간으로 클라이언트에 스트리밍한다.

```typescript
// packages/opencode/src/server/server.ts — GET /event 엔드포인트
app.get("/event", async (c) => {
  return streamSSE(c, async (stream) => {
    const unsub = Bus.subscribeAll((event) => {
      stream.writeSSE({
        data: JSON.stringify(event),
        event: event.type,
      })
    })

    // 30초 간격 하트비트
    const heartbeat = setInterval(() => {
      stream.writeSSE({ data: "", event: "heartbeat" })
    }, 30000)

    // 연결 종료 시 정리
    stream.onAbort(() => {
      unsub()
      clearInterval(heartbeat)
    })
  })
})
```

클라이언트(Web/Desktop)는 `EventSource`를 통해 SSE 스트림에 연결하고, 이벤트 타입으로 필터링하여 UI를 업데이트한다.

## 사용 예시

### 세션 업데이트 구독

```typescript
import { Bus } from "./bus"
import { Session } from "./session"

// 세션 업데이트 수신
const unsub = Bus.subscribe(Session.Event.Updated, (props) => {
  console.log(`세션 "${props.info.title}" 업데이트됨`)
})

// 구독 해제
unsub()
```

### 권한 요청 처리

```typescript
import { PermissionNext } from "./permission/next"

Bus.subscribe(PermissionNext.Event.Asked, (props) => {
  // UI에 권한 요청 다이얼로그 표시
  showPermissionDialog(props.request)
})
```

## 참고 문서

- [03. 핵심 패턴](./03-core-patterns.md) — BusEvent 패턴 설명
- [05. 세션 생명주기](./05-session-lifecycle.md) — 세션 이벤트 흐름
- [11. 서버 API](./11-server-api.md) — SSE 엔드포인트 상세
