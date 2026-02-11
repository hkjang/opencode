# 22. 명령어와 스킬 시스템

이 문서는 OpenCode의 명령어(Command)와 스킬(Skill) 시스템을 설명한다. 슬래시 명령어, 커스텀 명령어, 외부 스킬 검색을 다룬다.

**소스 파일:**
- `packages/opencode/src/command/index.ts` — 명령어 레지스트리
- `packages/opencode/src/skill/skill.ts` — 스킬 관리
- `packages/opencode/src/skill/discovery.ts` — 원격 스킬 다운로드

## 명령어 시스템 (Command)

### 개요

명령어는 사용자가 `/명령어이름`으로 호출하는 기능이다. 내장 명령어, 설정 명령어, MCP 프롬프트, 스킬이 모두 통합 명령어 인터페이스로 제공된다.

### Command.Info 스키마

```typescript
// packages/opencode/src/command/index.ts
export const Info = z.object({
  name: z.string(),           // 명령어 이름
  description: z.string(),    // 설명
  template: z.string(),       // 프롬프트 템플릿
  agent: z.string().optional(), // 사용할 에이전트
  model: z.string().optional(), // 사용할 모델
  // 소스 정보
})
```

### 명령어 소스 (우선순위)

`Command.list()`는 4가지 소스에서 명령어를 수집한다:

```
1. 내장 명령어 (Built-in)
   ├── init   — AGENTS.md 초기화 프롬프트
   └── review — 변경사항 리뷰 프롬프트

2. 설정 명령어 (Config)
   └── opencode.json의 commands 필드
   └── 설정 디렉터리의 command/*.md 파일

3. MCP 프롬프트
   └── 연결된 MCP 서버의 프롬프트

4. 스킬
   └── Skill.all()에서 가져온 스킬 목록
```

### 템플릿 변수

명령어 템플릿은 `$1`, `$2`, `$ARGUMENTS` 등의 변수를 지원한다:

```markdown
<!-- ~/.config/opencode/command/deploy.md -->
---
agent: build
---

$1 환경에 프로젝트를 배포합니다.
대상 브랜치: $2
추가 인자: $ARGUMENTS
```

```typescript
// Command.hints()로 변수 추출
const hints = Command.hints("$1 환경에 $2 브랜치를 배포")
// hints = ["$1", "$2"]
```

### 이벤트

```typescript
export const Event = {
  Executed: BusEvent.define("command.executed", z.object({
    name: z.string(),
  })),
}
```

### 커스텀 명령어 생성

**JSON 설정:**
```jsonc
// opencode.json
{
  "commands": {
    "test": {
      "template": "프로젝트의 테스트를 실행하고 결과를 분석하세요.",
      "agent": "build"
    }
  }
}
```

**Markdown 파일:**
```markdown
<!-- ~/.config/opencode/command/test.md -->
---
agent: build
model: anthropic/claude-sonnet-4-20250514
---

프로젝트의 테스트를 실행하고 결과를 분석하세요.
실패한 테스트가 있으면 원인을 파악하고 수정안을 제시하세요.
```

사용: `/test`

## 스킬 시스템 (Skill)

### 개요

스킬(Skill)은 재사용 가능한 프롬프트 패키지이다. `SKILL.md` 파일과 관련 파일들로 구성되며, 로컬 디렉터리나 원격 URL에서 검색된다.

### Skill.Info 스키마

```typescript
// packages/opencode/src/skill/skill.ts
export const Info = z.object({
  name: z.string(),
  description: z.string(),
  location: z.string(),   // 파일 경로 또는 URL
  content: z.string(),    // SKILL.md 내용
})
```

### 스킬 검색 순서

`Skill.all()`은 다음 위치를 순서대로 검색한다:

```
1. 전역 외부 스킬
   ├── ~/.claude/skills/
   └── ~/.agents/skills/

2. 프로젝트 레벨 외부 스킬
   ├── .claude/skills/
   └── .agents/skills/

3. .opencode 디렉터리 스킬
   └── .opencode/skill/

4. 설정 경로
   └── config.skills.paths[]   (~/로 시작하면 홈 디렉터리 확장)

5. 원격 URL (자동 다운로드)
   └── config.skills.urls[]
```

**비활성화:** `OPENCODE_DISABLE_EXTERNAL_SKILLS` 환경 변수로 외부 스킬을 비활성화할 수 있다.

### SKILL.md 형식

```markdown
---
name: code-review
description: 코드 리뷰를 수행합니다
---

# 코드 리뷰 스킬

다음 기준으로 코드를 리뷰하세요:
1. 타입 안전성
2. 에러 처리
3. 성능
```

스킬 디렉터리의 관련 파일(최대 10개)이 함께 로드되어 스킬 컨텍스트를 보강한다.

### 원격 스킬 다운로드 (Discovery)

```typescript
// packages/opencode/src/skill/discovery.ts
export namespace Discovery {
  // 다운로드 캐시 디렉터리
  export function dir(): string  // Global.Path.cache/skills

  // 원격 인덱스에서 스킬 다운로드
  export async function pull(url: string): Promise<void>
}
```

원격 스킬은 `Global.Path.cache/skills` 디렉터리에 캐싱되어 오프라인에서도 사용할 수 있다.

### 에러 처리

```typescript
// 스킬을 찾을 수 없음
export const InvalidError = NamedError.create("SkillInvalid", z.object({
  name: z.string(),
}))

// SKILL.md의 name과 디렉터리명 불일치
export const NameMismatchError = NamedError.create("SkillNameMismatch", z.object({
  expected: z.string(),
  actual: z.string(),
}))
```

### 스킬 도구 연동

스킬은 `SkillTool`을 통해 에이전트가 직접 호출할 수도 있다:

```typescript
// packages/opencode/src/tool/skill.ts
// 에이전트가 /skill 도구로 스킬을 로드
// 스킬 디렉터리의 파일 목록과 함께 내용을 반환
```

## 명령어 vs 스킬 vs MCP 프롬프트

| 기능 | 명령어 | 스킬 | MCP 프롬프트 |
|------|--------|------|-------------|
| 호출 방법 | `/이름` | `/이름` 또는 도구 | `/이름` |
| 정의 위치 | JSON/Markdown | SKILL.md | MCP 서버 |
| 인자 지원 | `$1`, `$ARGUMENTS` | 프롬프트 내 컨텍스트 | MCP 스키마 |
| 에이전트 지정 | 가능 | 불가 | 불가 |
| 관련 파일 포함 | 불가 | 최대 10개 | 불가 |
| 원격 소스 | 불가 | URL 지원 | MCP 서버 |

## 참고 문서

- [04. 에이전트 시스템](./04-agent-system.md) — 명령어의 에이전트 지정
- [07. 도구 시스템](./07-tool-system.md) — SkillTool
- [10. 설정 시스템](./10-config-system.md) — 명령어/스킬 설정
- [13. MCP 통합](./13-mcp-integration.md) — MCP 프롬프트
- [15. CLI와 명령어](./15-cli-and-commands.md) — 슬래시 명령어 사용
