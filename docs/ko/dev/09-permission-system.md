# 09. 권한 시스템

이 문서는 OpenCode의 권한(Permission) 시스템을 설명한다. 도구 실행 시 사용자 승인을 관리하는 핵심 보안 모듈이다.

## 개요

권한 시스템(`PermissionNext`)은 도구가 파일 시스템, 셸, 네트워크 등에 접근할 때 허용/거부/질의(allow/deny/ask)를 결정한다. 룰셋(Ruleset)으로 정의된 규칙을 패턴 매칭하여 자동 판정하거나, 사용자에게 직접 물어본다.

**소스 파일:** `packages/opencode/src/permission/next.ts`

## 핵심 타입

```typescript
// 권한 동작
export const Action = z.enum(["allow", "deny", "ask"])

// 단일 규칙
export const Rule = z.object({
  permission: z.string(),  // 권한 이름 (예: "file.read", "bash")
  pattern: z.string(),     // 와일드카드 패턴 (예: "src/**", "*")
  action: Action,          // "allow" | "deny" | "ask"
})

// 룰셋 = 규칙 배열
export const Ruleset = z.array(Rule)

// 권한 요청
export const Request = z.object({
  id: z.string(),
  sessionID: z.string(),
  permission: z.string(),
  patterns: z.array(z.string()),
  metadata: z.record(z.unknown()),
  // 도구 컨텍스트 정보
})

// 사용자 응답
export const Reply = z.enum(["once", "always", "reject"])
```

## 권한 평가 흐름

### 1. 도구 실행 시 권한 요청

도구가 실행될 때, 해당 작업에 필요한 권한을 `PermissionNext.ask()`로 요청한다.

```typescript
// 예: Bash 도구에서 명령 실행 전
await PermissionNext.ask({
  sessionID,
  permission: "bash",
  patterns: ["npm install"],
  metadata: { command: "npm install" },
})
```

### 2. 룰셋 평가

`evaluate()` 함수가 룰셋을 순서대로 순회하며 매칭되는 규칙을 찾는다.

```typescript
export function evaluate(
  permission: string,
  pattern: string,
  ...rulesets: Ruleset[]
): Action
```

**평가 순서:**
1. 시스템 기본 룰셋 (에이전트별)
2. 에이전트 설정 룰셋
3. 사용자 설정 룰셋
4. 사용자가 "always"로 승인한 룰셋

매칭되는 규칙이 없으면 기본값 `"ask"`를 반환한다.

### 3. 패턴 매칭

와일드카드(Wildcard) 패턴을 지원한다:
- `*` — 모든 값에 매칭
- `src/**` — src 하위 모든 경로
- `~/` — 홈 디렉터리 확장
- `$HOME/` — 환경 변수 확장

### 4. 사용자 상호작용

`"ask"`로 평가된 경우:

```
ask() 호출
  └── Event.Asked 이벤트 발행
       └── 클라이언트에서 다이얼로그 표시
            └── 사용자 선택: once / always / reject
                 └── reply() 호출
                      ├── "once": 이번 요청만 허용
                      ├── "always": 룰셋에 영구 추가
                      └── "reject": 거부 (피드백 포함 가능)
```

## 에러 클래스

```typescript
// 사용자가 메시지 없이 거부
export const RejectedError = NamedError.create("PermissionRejected", ...)

// 사용자가 피드백과 함께 거부
export const CorrectedError = NamedError.create("PermissionCorrected", z.object({
  message: z.string(),  // 사용자의 수정 메시지
}))

// 설정 규칙에 의해 자동 거부
export const DeniedError = NamedError.create("PermissionDenied", ...)
```

프로세서(Processor)는 이 에러들을 구분하여 처리한다:
- `RejectedError` / `CorrectedError` → 도구 실행 중단, "blocked" 상태
- `DeniedError` → 자동으로 에이전트에 알림

## 권한 계층

에이전트별로 3단계 권한 계층이 존재한다:

```
시스템 기본 (agent.ts에 하드코딩)
  └── 에이전트 설정 (config.agent.*.permission)
       └── 사용자 설정 (opencode.json permission)
```

예를 들어, `plan` 에이전트는 시스템 기본으로 파일 쓰기가 거부되어 있다:
- 시스템 기본: `{ permission: "file.write", pattern: "*", action: "deny" }`
- 사용자가 설정으로 이를 `"allow"`로 재정의할 수 있음

## 주요 함수

### ask()

```typescript
export async function ask(input: {
  sessionID: string
  permission: string
  patterns: string[]
  metadata: Record<string, unknown>
}): Promise<void>
```

룰셋을 평가하여 `"allow"`면 즉시 반환, `"deny"`면 `DeniedError` 발생, `"ask"`면 이벤트를 발행하고 사용자 응답을 대기한다.

### reply()

```typescript
export async function reply(input: {
  id: string           // 요청 ID
  reply: Reply         // "once" | "always" | "reject"
  message?: string     // 거부 시 피드백
}): Promise<void>
```

사용자 응답을 처리한다. `"always"`인 경우 같은 세션의 관련 대기 중인 요청도 함께 승인한다.

### fromConfig()

```typescript
export function fromConfig(permission: Config.Permission): Ruleset
```

Config의 권한 설정을 Ruleset 형식으로 변환한다.

### disabled()

```typescript
export function disabled(tools: string[], ruleset: Ruleset): Set<string>
```

`"deny"` 규칙에 의해 비활성화된 도구 목록을 반환한다. LLM에 전달할 도구 목록에서 제외할 때 사용한다.

## 설정 예시

```jsonc
// opencode.json
{
  "permission": {
    "bash": {
      "allow": ["npm test", "npm run build"],
      "deny": ["rm -rf *"],
      "ask": ["*"]
    },
    "file.write": {
      "allow": ["src/**"],
      "deny": ["node_modules/**"],
      "ask": ["*"]
    }
  }
}
```

## 상태 관리

`Instance.state()`를 사용하여 다음을 관리한다:
- **pending**: 대기 중인 권한 요청 `Map<string, Request>`
- **approved**: 사용자가 "always"로 승인한 룰셋 (Storage에 영속)

## 참고 문서

- [04. 에이전트 시스템](./04-agent-system.md) — 에이전트별 권한 설정
- [07. 도구 시스템](./07-tool-system.md) — 도구가 권한을 요청하는 방법
- [10. 설정 시스템](./10-config-system.md) — 권한 설정 구조
