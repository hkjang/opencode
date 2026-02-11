# 11. 서버 API

이 문서는 OpenCode의 HTTP 서버를 설명한다. Hono 프레임워크 기반으로 REST API, SSE 이벤트, 정적 파일 제공을 담당한다.

**소스 파일:** `packages/opencode/src/server/server.ts`

## 개요

OpenCode 서버는 Hono 웹 프레임워크 위에 구축되며, Bun의 HTTP 서버로 실행된다. 기본 포트는 4096이다. 모든 클라이언트(TUI, Web, Desktop)는 이 서버와 HTTP/SSE로 통신한다.

```bash
# 서버 시작
bun dev serve              # 기본 포트 4096
bun dev serve --port 8080  # 포트 지정
```

## 라우트 구조

### 핵심 라우트

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/path` | 주요 경로 반환 (home, state, config, worktree, directory) |
| `GET` | `/vcs` | VCS 정보 (Git 브랜치 등) |
| `GET` | `/event` | SSE 이벤트 스트림 |
| `GET` | `/doc` | OpenAPI 문서 |
| `GET` | `/command` | 사용 가능한 명령어 목록 |
| `GET` | `/agent` | 사용 가능한 에이전트 목록 |
| `GET` | `/skill` | 사용 가능한 스킬 목록 |
| `GET` | `/lsp` | LSP 서버 상태 |
| `GET` | `/formatter` | 포매터 상태 |
| `POST` | `/log` | 로그 항목 기록 |
| `POST` | `/instance/dispose` | 인스턴스 정리 |
| `PUT` | `/auth/:providerID` | 인증 설정 |
| `DELETE` | `/auth/:providerID` | 인증 삭제 |

### 도메인별 라우트

| 접두어 | 설명 | 소스 |
|--------|------|------|
| `/global` | 전역 설정/상태 | GlobalRoutes |
| `/project` | 프로젝트 관리 | ProjectRoutes |
| `/session` | 세션 CRUD, 메시지 | SessionRoutes |
| `/permission` | 권한 요청/응답 | PermissionRoutes |
| `/question` | 사용자 질문 | QuestionRoutes |
| `/provider` | 프로바이더/모델 | ProviderRoutes |
| `/config` | 설정 읽기/쓰기 | ConfigRoutes |
| `/mcp` | MCP 서버 관리 | McpRoutes |
| `/pty` | 의사 터미널 | PtyRoutes |
| `/tui` | TUI 전용 | TuiRoutes |
| `/experimental` | 실험적 기능 | ExperimentalRoutes |
| `/` | 정적 파일/프록시 | FileRoutes |

## SSE 이벤트 스트림

`GET /event` 엔드포인트는 실시간 이벤트를 클라이언트에 스트리밍한다.

```typescript
// 서버 측
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

    stream.onAbort(() => {
      unsub()
      clearInterval(heartbeat)
    })
  })
})
```

**클라이언트 측:**
```typescript
const events = new EventSource("http://localhost:4096/event")
events.addEventListener("session.updated", (e) => {
  const data = JSON.parse(e.data)
  // UI 업데이트
})
```

## 미들웨어

### 인증

`OPENCODE_SERVER_PASSWORD` 환경 변수가 설정된 경우 Basic 인증이 활성화된다.

### CORS

다음 도메인에서의 교차 출처 요청을 허용한다:
- `localhost` (개발)
- `tauri://localhost` (데스크톱 앱)
- `*.opencode.ai` (웹 서비스)

### 요청 로깅

모든 요청의 메서드, 경로, 응답 시간을 로깅한다.

### 에러 처리

```typescript
// NamedError를 적절한 HTTP 상태 코드로 변환
app.onError((err, c) => {
  if (err instanceof NamedError) {
    return c.json({ error: err.name, properties: err.properties }, 400)
  }
  return c.json({ error: "InternalError" }, 500)
})
```

## OpenAPI 스펙

`Server.openapi()`는 등록된 모든 라우트의 OpenAPI 3.0 스펙을 생성한다. 이 스펙은 JavaScript SDK(`packages/sdk/js`)의 자동 생성에 사용된다.

```bash
# OpenAPI 스펙 확인
curl http://localhost:4096/doc
```

## mDNS 서비스 발견

서버는 mDNS를 통해 로컬 네트워크에서 자동으로 검색할 수 있다. 이는 데스크톱 앱이 실행 중인 서버를 찾는 데 사용된다.

## 인스턴스 격리

서버는 디렉터리 파라미터로 인스턴스를 격리한다. 각 요청은 `Instance.provide()`를 통해 특정 프로젝트 컨텍스트에서 실행된다.

## 참고 문서

- [01. 아키텍처 개요](./01-architecture-overview.md) — 서버 계층의 위치
- [08. 이벤트 버스](./08-bus-event-system.md) — SSE로 전달되는 이벤트
- [15. CLI와 명령어](./15-cli-and-commands.md) — `serve` 명령어
