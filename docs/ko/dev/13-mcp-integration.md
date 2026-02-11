# 13. MCP 통합

이 문서는 OpenCode의 MCP(Model Context Protocol) 통합을 설명한다. 외부 MCP 서버에서 도구, 프롬프트, 리소스를 가져와 에이전트에 제공한다.

**소스 파일:** `packages/opencode/src/mcp/index.ts`

## 개요

MCP는 LLM 애플리케이션이 외부 도구 서버와 상호작용하기 위한 표준 프로토콜이다. OpenCode는 MCP 클라이언트를 구현하여 다양한 MCP 서버와 연결하고, 서버가 제공하는 도구를 내장 도구처럼 사용할 수 있게 한다.

## 연결 타입

### 로컬 (stdio)

로컬 프로세스를 실행하고 stdin/stdout으로 통신한다.

```jsonc
// opencode.json
{
  "mcp": {
    "github": {
      "type": "local",
      "command": ["npx", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_TOKEN": "{env:GITHUB_TOKEN}"
      },
      "timeout": 30000  // 밀리초 (기본: 30초)
    }
  }
}
```

### 원격 (HTTP/SSE)

원격 서버에 HTTP 또는 SSE로 연결한다.

```jsonc
{
  "mcp": {
    "my-server": {
      "type": "remote",
      "url": "https://mcp.example.com/sse",
      "headers": {
        "Authorization": "Bearer {env:MCP_TOKEN}"
      }
    }
  }
}
```

### OAuth

OAuth 인증이 필요한 원격 서버에 연결한다.

```jsonc
{
  "mcp": {
    "my-oauth-server": {
      "type": "oauth",
      "url": "https://mcp.example.com",
      "clientID": "optional-client-id"
    }
  }
}
```

## 연결 상태

각 MCP 서버의 연결 상태를 추적한다:

```typescript
export const Status = z.discriminatedUnion("status", [
  z.object({ status: z.literal("connected") }),
  z.object({ status: z.literal("disabled") }),
  z.object({ status: z.literal("failed"), error: z.string() }),
  z.object({ status: z.literal("needs_auth"), url: z.string() }),
  z.object({ status: z.literal("needs_client_registration") }),
])
```

## 주요 함수

### MCP.connect()

```typescript
export async function connect(name: string): Promise<void>
```

설정된 MCP 서버에 연결한다. 연결 타입에 따라:
- `local`: 자식 프로세스 생성 + stdio 트랜스포트
- `remote`: StreamableHTTP 또는 SSE 트랜스포트
- `oauth`: OAuth 흐름 시작 (필요 시)

### MCP.tools()

```typescript
export function tools(): Record<string, Tool>
```

연결된 모든 MCP 서버의 도구를 AI SDK 형식으로 변환하여 반환한다.

### MCP.status()

```typescript
export function status(): Record<string, Status>
```

모든 MCP 서버의 현재 연결 상태를 반환한다.

### MCP.resources()

```typescript
export function resources(): Resource[]
```

연결된 MCP 서버의 리소스(파일, 데이터 등) 목록을 반환한다.

### MCP.prompts()

```typescript
export function prompts(): Prompt[]
```

연결된 MCP 서버의 프롬프트 템플릿 목록을 반환한다.

## 도구 변환

MCP 서버의 도구 정의를 AI SDK의 `Tool` 형식으로 변환한다:

```typescript
// 내부 변환 로직
function convertMcpTool(mcpTool, client, timeout) {
  return {
    description: mcpTool.description,
    parameters: jsonSchemaToZod(mcpTool.inputSchema),
    async execute(args) {
      const result = await client.callTool({
        name: mcpTool.name,
        arguments: args,
      }, { timeout })
      return result.content
    },
  }
}
```

**도구 이름 정규화:**
MCP 도구 이름은 `{서버이름}_{도구이름}` 형태로 정규화되어 내장 도구와 이름 충돌을 방지한다.

## OAuth 흐름

OAuth 타입 MCP 서버의 인증 흐름:

```
1. MCP.startAuth(name)
   └── OAuth 서버에 인증 요청
       └── 동적 클라이언트 등록 (clientID가 없는 경우)
           └── 인증 URL 반환

2. 브라우저에서 인증 URL 열기
   └── 사용자가 승인

3. MCP.finishAuth(name, code)
   └── 인증 코드로 토큰 교환
       └── 토큰 저장 + MCP 연결
```

### 토큰 관리

```typescript
// 토큰 상태 확인
MCP.hasStoredTokens(name)  // 저장된 토큰 존재 여부
MCP.getAuthStatus(name)     // "authenticated" | "expired" | "not_authenticated"
MCP.removeAuth(name)        // 토큰 삭제
```

## 이벤트

```typescript
// MCP 서버의 도구 목록이 변경되었을 때
MCP.ToolsChanged = BusEvent.define("mcp.tools_changed", z.object({
  name: z.string(),
}))

// OAuth 인증 시 브라우저 열기 실패
MCP.BrowserOpenFailed = BusEvent.define("mcp.browser_open_failed", z.object({
  url: z.string(),
}))
```

## 알림 처리

MCP 서버로부터의 알림(Notification)을 처리한다:
- `tools/list_changed` → 도구 목록 새로고침 + `ToolsChanged` 이벤트 발행
- 기타 알림 → 로깅

## 상태 관리

`Instance.state()`를 사용하여 관리:
- `status: Map<string, Status>` — 서버별 연결 상태
- `clients: Map<string, Client>` — 서버별 MCP 클라이언트

인스턴스 해제 시 모든 MCP 클라이언트 연결이 종료된다.

## 참고 문서

- [07. 도구 시스템](./07-tool-system.md) — MCP 도구가 ToolRegistry에 통합되는 방식
- [10. 설정 시스템](./10-config-system.md) — MCP 설정 구조
- [12. 플러그인 시스템](./12-plugin-system.md) — 플러그인과 MCP의 차이
