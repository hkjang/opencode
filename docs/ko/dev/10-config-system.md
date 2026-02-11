# 10. 설정 시스템

이 문서는 OpenCode의 설정(Config) 시스템을 설명한다. 다중 소스에서 설정을 로딩하고 우선순위에 따라 병합하는 구조이다.

**소스 파일:** `packages/opencode/src/config/config.ts`

## 개요

설정 시스템은 JSONC(JSON with Comments) 형식을 지원하며, 환경 변수 참조(`{env:VAR}`)와 파일 참조(`{file:path}`)를 확장한다. 여러 경로에서 설정을 로드하고, 정의된 우선순위에 따라 병합한다.

## 설정 로딩 우선순위

나중에 로드되는 설정이 이전 설정을 덮어쓴다 (숫자가 클수록 높은 우선순위):

```
1. Remote (.well-known/opencode)     ← 조직 기본값 (최저)
2. Global (~/.config/opencode/)       ← 사용자 전역 설정
3. Custom (OPENCODE_CONFIG 환경변수)  ← 사용자 지정 경로
4. Project (opencode.json/jsonc)      ← 프로젝트 로컬 설정
5. .opencode 디렉터리                 ← 프로젝트 디렉터리 설정
6. Inline (OPENCODE_CONFIG_CONTENT)  ← 인라인 JSON 설정
7. Managed (엔터프라이즈)              ← 관리 설정 (최고)
```

**특수 규칙:** `plugins`와 `instructions` 배열은 덮어쓰지 않고 **연결(concatenate)**된다.

## Config.Info 스키마

주요 설정 필드:

```typescript
export const Info = z.object({
  // 모델 설정
  provider: z.string().optional(),     // 기본 프로바이더
  model: z.string().optional(),        // 기본 모델

  // 에이전트 설정
  agent: z.record(Agent).optional(),   // 에이전트별 설정

  // 권한 설정
  permission: Permission.optional(),   // 도구별 권한 규칙

  // MCP 서버
  mcp: z.record(Mcp).optional(),       // MCP 서버 설정

  // 플러그인
  plugins: z.array(z.string()),        // 플러그인 패키지 목록

  // 지시사항
  instructions: z.array(z.string()),   // 시스템 프롬프트 추가 지시

  // 키바인드
  keybinds: Keybinds.optional(),       // 80+ 커스텀 키바인딩

  // 실험적 기능
  experimental: z.object({
    batch_tool: z.boolean().optional(),
    // ...
  }).optional(),

  // 세션 압축
  compaction: Compaction.optional(),

  // 스킬
  skills: Skills.optional(),

  // 프로바이더 목록
  providers: z.record(Provider).optional(),
})
```

## 주요 함수

### Config.get()

현재 로드된 설정을 반환한다.

```typescript
export function get(): Config.Info
```

내부적으로 `Instance.state()`를 통해 프로젝트별로 설정을 캐싱한다. 최초 호출 시 모든 소스에서 설정을 로드하고 병합한다.

### Config.getGlobal()

전역 사용자 설정을 반환한다 (`~/.config/opencode/config.json`).

```typescript
export function getGlobal(): Config.Info
```

### Config.update()

프로젝트 설정 파일(`opencode.json`)을 업데이트한다.

```typescript
export async function update(config: Partial<Config.Info>): Promise<void>
```

### Config.updateGlobal()

전역 설정 파일을 JSONC 형식을 보존하며 업데이트한다.

```typescript
export async function updateGlobal(config: Partial<Config.Info>): Promise<void>
```

### Config.directories()

설정 파일을 스캔한 모든 디렉터리 경로를 반환한다. 커스텀 도구, 에이전트, 명령어 파일을 탐색할 때 사용한다.

```typescript
export function directories(): string[]
```

## 설정 파일 형식

### 기본 설정

```jsonc
// opencode.json
{
  // 기본 프로바이더와 모델
  "provider": "anthropic",
  "model": "claude-sonnet-4-20250514",

  // 에이전트 커스터마이즈
  "agent": {
    "build": {
      "model": "anthropic/claude-sonnet-4-20250514",
      "prompt": "항상 한국어로 응답하세요"
    }
  },

  // 권한 규칙
  "permission": {
    "bash": {
      "allow": ["npm test"],
      "ask": ["*"]
    }
  },

  // MCP 서버
  "mcp": {
    "github": {
      "type": "local",
      "command": ["npx", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "{env:GITHUB_TOKEN}" }
    }
  }
}
```

### 환경 변수 참조

```jsonc
{
  "mcp": {
    "myserver": {
      "env": {
        // {env:변수명} 형식으로 환경 변수 참조
        "API_KEY": "{env:MY_API_KEY}"
      }
    }
  }
}
```

### 파일 참조

```jsonc
{
  "instructions": [
    // {file:경로} 형식으로 파일 내용을 인라인
    "{file:./CODING_GUIDELINES.md}"
  ]
}
```

## MCP 설정 타입

MCP 서버는 세 가지 타입을 지원한다:

```typescript
// 로컬 프로세스
export const McpLocal = z.object({
  type: z.literal("local"),
  command: z.array(z.string()),  // 실행 명령어
  env: z.record(z.string()).optional(),
  timeout: z.number().optional(),
})

// 원격 HTTP/SSE
export const McpRemote = z.object({
  type: z.literal("remote"),
  url: z.string(),
  headers: z.record(z.string()).optional(),
  timeout: z.number().optional(),
})

// OAuth 인증 원격
export const McpOAuth = z.object({
  type: z.literal("oauth"),
  url: z.string(),
  clientID: z.string().optional(),
})

export const Mcp = z.discriminatedUnion("type", [McpLocal, McpRemote, McpOAuth])
```

## 커스텀 에이전트 / 명령어 / 모드

설정 디렉터리 내에서 Markdown 파일로 커스텀 에이전트, 명령어, 모드를 정의할 수 있다:

```
~/.config/opencode/
├── config.json         # 전역 설정
├── agent/              # 또는 agents/
│   └── reviewer.md     # 커스텀 에이전트
├── command/            # 또는 commands/
│   └── deploy.md       # 커스텀 슬래시 명령어
└── mode/               # 또는 modes/
    └── korean.md       # 커스텀 모드
```

에이전트/명령어 Markdown 파일은 YAML 프론트매터(Frontmatter)를 지원한다:

```markdown
---
model: anthropic/claude-sonnet-4-20250514
tools:
  bash: true
  file.read: true
---

이 에이전트는 코드 리뷰에 특화되어 있습니다.
항상 한국어로 응답하세요.
```

## 에러 처리

```typescript
// JSON 파싱 에러
export const JsonError = NamedError.create("ConfigJson", z.object({
  path: z.string(),
}))

// 스키마 검증 에러
export const InvalidError = NamedError.create("ConfigInvalid", z.object({
  path: z.string(),
  errors: z.array(z.any()),
}))

// 디렉터리 오타 감지 (예: .opencode 대신 .opecode)
export const ConfigDirectoryTypoError = NamedError.create(...)
```

## 레거시 마이그레이션

이전 설정 형식은 자동으로 변환된다:
- `tools` → `permission` (도구별 권한으로 이동)
- `mode` → `agent` (모드 개념이 에이전트로 통합)

## 참고 문서

- [03. 핵심 패턴](./03-core-patterns.md) — 설정 우선순위 패턴
- [04. 에이전트 시스템](./04-agent-system.md) — 에이전트 설정
- [09. 권한 시스템](./09-permission-system.md) — 권한 설정 구조
- [12. 플러그인 시스템](./12-plugin-system.md) — 플러그인 설정
- [13. MCP 통합](./13-mcp-integration.md) — MCP 서버 설정
