# 04. 에이전트 시스템

이 문서는 OpenCode의 에이전트(Agent) 시스템을 설명한다. 에이전트는 특정 역할에 맞는 모델, 도구, 시스템 프롬프트, 권한을 정의한다.

**소스 파일:** `packages/opencode/src/agent/agent.ts`

## 개요

에이전트는 "어떤 LLM을, 어떤 도구와 권한으로, 어떤 지시사항에 따라 실행할 것인가"를 결정하는 설정 단위이다. OpenCode에는 6개의 내장 에이전트가 있으며, 사용자가 설정으로 커스텀 에이전트를 추가할 수 있다.

## 내장 에이전트

| 에이전트 | 모드 | 설명 |
|---------|------|------|
| `build` | primary | 기본 코딩 에이전트. 코드 읽기/쓰기, 셸 실행 |
| `plan` | primary | 읽기 전용 에이전트. 분석 및 계획 수립 (쓰기 금지) |
| `general` | primary | 범용 에이전트 |
| `explore` | subagent | 코드베이스 탐색 전용 (쓰기 금지) |
| `compaction` | subagent | 컨텍스트 초과 시 메시지 압축 |
| `title` | subagent | 세션 제목 자동 생성 |
| `summary` | subagent | 세션 요약 생성 |

**모드 설명:**
- `primary`: 사용자가 직접 선택할 수 있는 에이전트
- `subagent`: 시스템이 내부적으로 사용하는 에이전트
- `all`: 모든 컨텍스트에서 사용 가능

## Agent.Info 스키마

```typescript
export const Info = z.object({
  id: z.string(),                          // 에이전트 ID
  name: z.string(),                        // 표시 이름
  model: z.string().optional(),            // "providerID/modelID"
  prompt: z.string().optional(),           // 시스템 프롬프트
  tools: z.record(z.boolean()).optional(), // 도구 활성화/비활성화
  permission: Permission.optional(),       // 권한 룰셋
  options: z.record(z.any()).optional(),   // 모델 옵션
  mode: z.enum(["subagent", "primary", "all"]).optional(),
  // ...기타 필드
})
```

## 에이전트 검색 및 초기화

### Agent.get()

```typescript
export function get(agent: string): Agent.Info
```

에이전트 이름으로 설정을 검색한다. 내장 에이전트와 사용자 정의 에이전트를 모두 탐색한다.

### Agent.list()

```typescript
export function list(): Agent.Info[]
```

사용 가능한 모든 에이전트 목록을 반환한다.

### Agent.defaultAgent()

```typescript
export function defaultAgent(): Agent.Info
```

설정된 기본 에이전트를 반환한다 (일반적으로 `build`).

## 권한 병합

에이전트의 권한은 3단계로 병합된다:

```
시스템 기본 (agent.ts 하드코딩)
  │ permission.defaults
  └──▶ 에이전트 설정 (config.agent.*.permission)
        │ permission.agent
        └──▶ 사용자 설정 (opencode.json permission)
              │ permission.user
              └──▶ 최종 Ruleset
```

```typescript
// 내부 병합 로직
const ruleset = PermissionNext.merge(
  systemDefaults,        // 에이전트별 기본 권한
  agentPermission,       // 에이전트 설정 권한
  userPermission,        // 사용자 설정 권한
)
```

### plan 에이전트의 권한 예시

`plan` 에이전트는 읽기 전용이므로 시스템 기본으로 쓰기 관련 권한이 거부되어 있다:

```typescript
// 시스템 기본 (하드코딩)
const planDefaults = [
  { permission: "file.write", pattern: "*", action: "deny" },
  { permission: "bash",       pattern: "*", action: "deny" },
  { permission: "mcp.*",      pattern: "*", action: "deny" },
]
```

## 시스템 프롬프트

에이전트의 시스템 프롬프트는 다음 소스에서 조합된다:

1. **에이전트 기본 프롬프트**: `agent.prompt` 필드
2. **프로바이더 프롬프트**: 프로바이더별 추가 지시사항
3. **사용자 커스텀 프롬프트**: `config.agent.*.prompt`
4. **사용자 메시지의 system 필드**: 메시지별 추가 지시

```typescript
// packages/opencode/src/session/llm.ts — 시스템 프롬프트 조합
const system = [
  agent.prompt,           // 에이전트 기본
  providerPrompt,         // 프로바이더별
  customSystem,           // 사용자 커스텀
  user.system,            // 메시지별
].filter(Boolean).join("\n")
```

## 커스텀 에이전트 생성

### 설정 파일로 생성

```jsonc
// opencode.json
{
  "agent": {
    "reviewer": {
      "model": "anthropic/claude-sonnet-4-20250514",
      "prompt": "코드 리뷰에 집중합니다. 한국어로 응답하세요.",
      "tools": {
        "bash": false,       // Bash 도구 비활성화
        "file.write": false  // 파일 쓰기 비활성화
      }
    }
  }
}
```

### Markdown 파일로 생성

```markdown
<!-- ~/.config/opencode/agent/reviewer.md -->
---
model: anthropic/claude-sonnet-4-20250514
tools:
  bash: false
  file.write: false
---

코드 리뷰에 집중합니다.
다음을 확인합니다:
- 타입 안전성
- 에러 처리
- 코드 스타일 가이드 준수
```

## Agent.generate()

AI를 사용하여 에이전트 설정을 자동 생성할 수도 있다:

```typescript
export async function generate(input: {
  description: string  // 에이전트 설명
}): Promise<Agent.Info>
```

## 참고 문서

- [05. 세션 생명주기](./05-session-lifecycle.md) — 에이전트가 세션에서 사용되는 방식
- [06. 프로바이더 시스템](./06-provider-system.md) — 에이전트의 모델 선택
- [07. 도구 시스템](./07-tool-system.md) — 에이전트별 도구 필터링
- [09. 권한 시스템](./09-permission-system.md) — 에이전트별 권한 설정
- [10. 설정 시스템](./10-config-system.md) — 에이전트 설정 구조
