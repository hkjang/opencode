# 03. 핵심 패턴

이 문서는 OpenCode 코드베이스 전반에서 반복적으로 사용되는 핵심 설계 패턴을 설명한다. 새 코드를 작성하거나 기존 코드를 수정할 때 이 패턴을 따라야 한다.

## Namespace 패턴

OpenCode의 모든 핵심 모듈은 TypeScript `export namespace`를 사용하여 관련 타입, 함수, 상수를 하나의 네임스페이스(Namespace)에 묶는다. 이 패턴은 모듈 경계를 명확하게 하고, 외부에서 `ModuleName.functionName()` 형태로 일관된 호출을 가능하게 한다.

```typescript
// packages/opencode/src/session/index.ts
export namespace Session {
  // Zod 스키마로 타입 정의
  export const Info = z.object({
    id: z.string(),
    projectID: z.string(),
    title: z.string(),
    // ...
  })
  export type Info = z.infer<typeof Info>

  // 이벤트 정의
  export const Event = {
    Updated: BusEvent.define("session.updated", z.object({ ... })),
  }

  // 함수
  export async function create(input: { ... }) { ... }
  export async function get(sessionID: string) { ... }
  export async function list() { ... }
}
```

**사용 시:**
```typescript
import { Session } from "./session"

const session = await Session.create({ ... })
const info: Session.Info = await Session.get(sessionID)
```

**적용 모듈:** `Agent`, `Provider`, `Session`, `Storage`, `Bus`, `BusEvent`, `Config`, `PermissionNext`, `MCP`, `Plugin`, `Tool`, `ToolRegistry`, `MessageV2` 등 거의 모든 모듈.

## Zod 스키마 패턴

모든 데이터 구조는 Zod 스키마로 정의하고, `z.infer`로 TypeScript 타입을 자동 추론한다. 이를 통해 런타임 검증과 타입 안전성을 동시에 확보한다.

```typescript
// packages/opencode/src/agent/agent.ts
export const Info = z.object({
  id: z.string(),
  name: z.string(),
  model: z.string().optional(),
  prompt: z.string().optional(),
  tools: z.record(z.boolean()).optional(),
  // ...
})
export type Info = z.infer<typeof Info>
```

**핵심 원칙:**
- `const`와 `type`에 같은 이름을 사용 (예: `Info`는 스키마이자 타입)
- 판별 유니온(Discriminated union)에는 `z.discriminatedUnion()` 사용
- 선택 필드는 `.optional()`, 기본값은 `.default()`
- 에러는 `NamedError.create()`로 타입 안전한 에러 클래스 생성

```typescript
// packages/opencode/src/session/message-v2.ts — 판별 유니온 예시
export const Part = z.discriminatedUnion("type", [
  TextPart,
  ToolPart,
  ReasoningPart,
  SnapshotPart,
  // ...
])
```

## Instance.state() 패턴

`Instance.state()`는 프로젝트 디렉터리에 종속된 지연 초기화(Lazy initialization) 상태를 생성한다. 각 모듈이 프로젝트별로 격리된 상태를 유지할 수 있게 하는 핵심 패턴이다.

```typescript
// packages/opencode/src/project/instance.ts
export function state<S>(
  init: () => Promise<S>,  // 초기화 함수
  dispose?: (state: S) => Promise<void>  // 정리 함수
): () => Promise<S>
```

**사용 예시:**

```typescript
// packages/opencode/src/config/config.ts
export namespace Config {
  // 디렉터리별로 설정을 한 번만 로드
  const state = Instance.state(async () => {
    // 설정 파일 로딩, 병합, 검증
    const config = await loadAndMerge()
    return { config, directories }
  })

  export function get() {
    return state().config
  }
}
```

**동작 방식:**
1. 최초 호출 시 `init()` 실행 후 결과를 캐싱
2. 이후 호출은 캐싱된 값 반환
3. `Instance.dispose()` 호출 시 `dispose()` 실행 후 캐시 제거
4. 같은 디렉터리에서 다시 호출하면 재초기화

**적용 모듈:** `Config.state()`, `Agent.state()`, `Provider.state()`, `ToolRegistry.state()`, `Bus.state()`, `PermissionNext.state()`, `MCP.state()`, `Plugin.state()`, `Storage.state()` 등.

## BusEvent 패턴

이벤트 버스(Event Bus)는 모듈 간 느슨한 결합을 위한 발행-구독(Pub/Sub) 시스템이다.

### 이벤트 정의

```typescript
// packages/opencode/src/bus/bus-event.ts
export namespace BusEvent {
  export function define<T extends z.ZodObject<any>>(
    type: string,
    schema: T
  ) {
    // 전역 레지스트리에 등록하고 타입 안전한 이벤트 정의 반환
    return { type, properties: schema }
  }
}
```

### 이벤트 선언 (모듈 내)

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
  }
}
```

### 이벤트 발행 및 구독

```typescript
// packages/opencode/src/bus/index.ts
export namespace Bus {
  // 발행 — 타입 안전
  export function publish<D extends BusEvent.Definition>(
    event: D,
    properties: z.infer<D["properties"]>
  ): Promise<void>

  // 구독
  export function subscribe<D extends BusEvent.Definition>(
    event: D,
    callback: (properties: z.infer<D["properties"]>) => void
  ): () => void  // 구독 해제 함수 반환
}

// 사용
Bus.publish(Session.Event.Updated, { info: session })
const unsub = Bus.subscribe(Session.Event.Updated, (props) => {
  console.log(props.info.title)
})
```

**GlobalBus:** `Bus.publish()`는 내부적으로 `GlobalBus`에도 이벤트를 전달한다. GlobalBus는 프로세스 간 통신(SSE 등)에 사용된다.

## NamedError 패턴

OpenCode는 커스텀 에러 클래스를 `NamedError.create()`로 생성한다.

```typescript
// 에러 정의
export const NotFoundError = NamedError.create("StorageNotFound", z.object({
  key: z.array(z.string()),
}))

// 에러 발생
throw new NotFoundError({ key: ["session", projectID, sessionID] })

// 에러 처리
import { NotFoundError } from "./storage"
if (e instanceof NotFoundError) {
  console.log(e.properties.key)
}
```

## 팩토리 함수 패턴

클래스 대신 팩토리 함수(Factory function)를 사용하여 객체를 생성한다. 이는 프로젝트 전반의 함수형 스타일과 일치한다.

```typescript
// packages/opencode/src/session/processor.ts
export namespace SessionProcessor {
  export function create(input: {
    sessionID: string
    agent: Agent.Info
    model: Provider.Model
    // ...
  }) {
    // 클로저로 상태 관리
    const toolcalls = new Map()
    let blocked = false

    return {
      async process(stream: LLM.StreamOutput) {
        // 스트림 이벤트 처리
        for await (const event of stream.fullStream) {
          // ...
        }
        return "continue" // | "compact" | "stop"
      }
    }
  }
}
```

## Tool.define() 패턴

도구는 `Tool.define()`으로 정의한다. Zod 스키마로 파라미터를 검증하고, 실행 함수를 래핑한다.

```typescript
// packages/opencode/src/tool/tool.ts
export namespace Tool {
  export function define<P extends z.ZodObject<any>, M>(options: {
    id: string
    init: () => Promise<{
      description: string
      parameters: P
      execute: (args: z.infer<P>, ctx: Context) => Promise<string>
    }>
  }): Tool.Info
}
```

**사용 예시:**

```typescript
const ReadTool = Tool.define({
  id: "read",
  init: async () => ({
    description: "Read a file from the filesystem",
    parameters: z.object({
      path: z.string().describe("파일 경로"),
    }),
    async execute(args, ctx) {
      const content = await Bun.file(args.path).text()
      return content
    },
  }),
})
```

## 설정 우선순위 패턴

Config는 여러 소스에서 설정을 로딩하고 우선순위에 따라 병합한다. 나중에 로드되는 설정이 앞의 설정을 덮어쓴다.

```
1. Remote (.well-known/opencode)     — 조직 기본값 (최저 우선순위)
2. Global (~/.config/opencode/)       — 사용자 전역 설정
3. Custom (OPENCODE_CONFIG 환경변수)  — 사용자 지정 경로
4. Project (opencode.json/jsonc)      — 프로젝트 로컬 설정
5. .opencode 디렉터리                 — 프로젝트 디렉터리 설정
6. Inline (OPENCODE_CONFIG_CONTENT)  — 인라인 설정
7. Managed (엔터프라이즈)              — 관리 설정 (최고 우선순위)
```

## 코딩 스타일 요약

[AGENTS.ko.md](../../../AGENTS.ko.md)의 스타일 가이드를 간략히 요약한다:

| 원칙 | 설명 |
|------|------|
| `const` 우선 | `let` 대신 `const`, 삼항 연산자, 조기 반환 |
| `else` 금지 | 조기 반환(early return)으로 대체 |
| 구조 분해 지양 | `obj.a` 사용, `const { a } = obj` 지양 |
| 단일 단어 명명 | 가능하면 짧은 단어: `session`, `agent` |
| 타입 추론 | 명시적 타입보다 `z.infer`, 추론에 의존 |
| 함수형 메서드 | `for` 루프보다 `map`, `filter`, `flatMap` |
| `.catch()` 우선 | `try/catch` 대신 `.catch()` |
| Bun API 활용 | `Bun.file()`, `Bun.$` 등 |

## 참고 문서

- [AGENTS.ko.md](../../../AGENTS.ko.md) — 전체 스타일 가이드
- [01. 아키텍처 개요](./01-architecture-overview.md) — 시스템 전체 구조
- [08. 이벤트 버스](./08-bus-event-system.md) — Bus 시스템 상세
- [10. 설정 시스템](./10-config-system.md) — Config 상세
- [14. 저장소와 상태](./14-storage-and-state.md) — Storage, Instance.state() 상세
