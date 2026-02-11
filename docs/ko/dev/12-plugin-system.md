# 12. 플러그인 시스템

이 문서는 OpenCode의 플러그인(Plugin) 시스템을 설명한다. 도구, 훅, 인증 등을 확장할 수 있는 모듈 시스템이다.

**소스 파일:** `packages/opencode/src/plugin/index.ts`

## 개요

플러그인 시스템은 세 종류의 플러그인을 지원한다:

1. **내부 플러그인(Internal)**: 코드베이스에 직접 포함 (CodexAuth, CopilotAuth, GitlabAuth)
2. **내장 플러그인(Builtin)**: npm 패키지로 기본 설치되는 플러그인
3. **사용자 플러그인**: `opencode.json`의 `plugins` 필드로 설치

## 플러그인 SDK

외부 플러그인은 `@opencode-ai/plugin` 패키지를 사용하여 개발한다.

```typescript
// packages/plugin/src/index.ts
import { definePlugin } from "@opencode-ai/plugin"

export default definePlugin({
  name: "my-plugin",
  hooks: {
    // 시스템 프롬프트 변환
    "experimental.chat.system.transform": async (input) => {
      return input.system + "\n추가 지시사항"
    },

    // 채팅 파라미터 설정
    "chat.params": async (input) => ({
      temperature: 0.7,
    }),

    // 커스텀 HTTP 헤더
    "chat.headers": async () => ({
      "X-Custom-Header": "value",
    }),
  },
})
```

## 플러그인 입력 컨텍스트

모든 플러그인은 초기화 시 다음 컨텍스트를 받는다:

```typescript
interface PluginInput {
  client: OpenCodeSDK      // OpenCode SDK 클라이언트 (서버와 통신)
  project: Project.Info     // 현재 프로젝트 정보
  worktree: string          // Git 워크트리 경로
  directory: string         // 작업 디렉터리
  serverUrl: string         // 로컬 서버 URL
  $: typeof Bun.$           // Bun 셸 실행 컨텍스트
}
```

## 플러그인 로딩 순서

```
1. 내부 플러그인 로드 (INTERNAL_PLUGINS)
   └── CodexAuth, CopilotAuth, GitlabAuth

2. 내장 플러그인 로드 (BUILTIN)
   └── OPENCODE_DISABLE_DEFAULT_PLUGINS로 비활성화 가능

3. 사용자 플러그인 로드 (config.plugins)
   └── npm 패키지 또는 file:// 경로
```

### npm 플러그인 설치

사용자 플러그인은 Config 디렉터리에 자동으로 설치된다:

```typescript
// 내부 동작
// 1. Config.directories()의 각 디렉터리에서 package.json 확인
// 2. @opencode-ai/plugin 의존성이 없으면 설치
// 3. 지정된 npm 패키지를 설치
// 4. import()로 플러그인 로드
```

### 파일 플러그인

로컬 파일도 플러그인으로 사용할 수 있다:

```jsonc
// opencode.json
{
  "plugins": [
    "my-npm-plugin",           // npm 패키지
    "my-plugin@1.2.3",         // 특정 버전
    "file:///path/to/plugin.ts" // 로컬 파일
  ]
}
```

## 훅 시스템

### Plugin.trigger()

```typescript
export async function trigger<Name, Input, Output>(
  name: Name,     // 훅 이름
  input: Input,   // 입력 데이터
  output: Output  // 기본 출력 (훅이 없으면 이 값 반환)
): Promise<Output>
```

### 사용 가능한 훅

| 훅 이름 | 설명 | 호출 시점 |
|---------|------|-----------|
| `experimental.chat.system.transform` | 시스템 프롬프트 변환 | LLM 호출 전 |
| `chat.params` | 채팅 파라미터 설정 | LLM 호출 전 |
| `chat.headers` | HTTP 헤더 추가 | LLM 요청 시 |
| `experimental.text.complete` | 텍스트 완료 처리 | LLM 텍스트 응답 완료 시 |
| `config` | 설정 변환 | 설정 로드 시 |

## 플러그인 도구

플러그인은 커스텀 도구를 정의할 수 있다:

```typescript
// @opencode-ai/plugin/tool
import { defineTool } from "@opencode-ai/plugin/tool"

export const myTool = defineTool({
  name: "my-tool",
  description: "커스텀 도구 설명",
  parameters: z.object({
    input: z.string(),
  }),
  async execute(args, ctx) {
    return `결과: ${args.input}`
  },
})
```

플러그인 도구는 `ToolRegistry`에 의해 내장 도구와 동일하게 취급된다.

## 상태 관리

`Instance.state()`를 사용하여 다음을 관리한다:
- SDK 클라이언트 인스턴스 (로컬 서버를 가리키는)
- 로드된 훅 배열
- 플러그인 입력 컨텍스트

인스턴스 해제 시 모든 플러그인 상태가 정리된다.

## 에러 처리

플러그인 로딩이나 실행 중 에러가 발생하면:
- 에러가 로깅됨
- Bus 이벤트로 에러 발행
- 다른 플러그인의 실행에는 영향 없음 (격리)

## 참고 문서

- [07. 도구 시스템](./07-tool-system.md) — 플러그인 도구가 레지스트리에 등록되는 방식
- [10. 설정 시스템](./10-config-system.md) — 플러그인 설정
- [13. MCP 통합](./13-mcp-integration.md) — MCP와 플러그인의 차이
