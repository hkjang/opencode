# 15. CLI와 명령어

이 문서는 OpenCode의 CLI(Command Line Interface) 구조와 명령어 시스템을 설명한다.

**소스 파일:**
- `packages/opencode/src/index.ts` — CLI 진입점(Entry point)
- `packages/opencode/src/cli/cmd/` — 하위 명령어
- `packages/opencode/src/cli/bootstrap.ts` — 부트스트랩

## CLI 구조

OpenCode CLI는 Yargs 라이브러리를 사용하여 구축되었다.

### 진입점

```typescript
// packages/opencode/src/index.ts
yargs(hideBin(process.argv))
  .command(RunCommand)       // 기본 명령어 (TUI 실행)
  .command(ServeCommand)     // API 서버
  .command(WebCommand)       // 웹 서버
  .command(AttachCommand)    // 세션 연결
  .command(AuthCommand)      // 인증 관리
  .command(AgentCommand)     // 에이전트 관리
  .command(McpCommand)       // MCP 서버 관리
  .command(ModelsCommand)    // 모델 목록
  .command(GenerateCommand)  // 코드 생성
  .command(ExportCommand)    // 세션 내보내기
  .command(ImportCommand)    // 세션 가져오기
  .command(GithubCommand)    // GitHub 연동
  .command(PrCommand)        // PR 관리
  .command(SessionCommand)   // 세션 관리
  .command(StatsCommand)     // 통계
  .command(UpgradeCommand)   // 업그레이드
  .command(UninstallCommand) // 제거
  .command(DebugCommand)     // 디버깅 유틸리티
  .command(AcpCommand)       // ACP 관련
  .command(TuiThreadCommand) // TUI 스레드
  // ...
```

### 전역 옵션

```bash
opencode --print-logs      # stderr로 로그 출력
opencode --log-level DEBUG # 로그 레벨 설정 (DEBUG, INFO, WARN, ERROR)
```

### 환경 변수

CLI 실행 시 자동 설정되는 환경 변수:
- `AGENT=1` — 에이전트 모드 표시
- `OPENCODE=1` — OpenCode 실행 중 표시

## 주요 명령어

### `opencode` (기본)

인자 없이 실행하면 TUI가 시작된다. 디렉터리를 인자로 주면 해당 디렉터리에서 실행된다.

```bash
opencode              # 현재 디렉터리에서 TUI 시작
opencode /path/to/dir # 지정 디렉터리에서 TUI 시작
```

### `opencode serve`

헤드리스 API 서버를 시작한다.

```bash
opencode serve              # 기본 포트 4096
opencode serve --port 8080  # 포트 지정
```

### `opencode web`

API 서버와 웹 인터페이스를 함께 시작한다.

```bash
opencode web
```

### `opencode attach`

실행 중인 서버에 TUI를 연결한다.

```bash
opencode attach http://localhost:4096
```

### `opencode auth`

프로바이더 인증을 관리한다.

```bash
opencode auth login anthropic   # Anthropic 인증
opencode auth logout anthropic  # 인증 삭제
opencode auth status            # 인증 상태 확인
```

### `opencode mcp`

MCP 서버를 관리한다.

```bash
opencode mcp list      # MCP 서버 목록
opencode mcp add       # MCP 서버 추가
opencode mcp remove    # MCP 서버 제거
```

### `opencode debug`

디버깅 유틸리티를 제공한다.

```bash
opencode debug config    # 현재 설정 출력
opencode debug agent     # 에이전트 정보
opencode debug file      # 파일 정보
opencode debug lsp       # LSP 상태
opencode debug snapshot  # 스냅샷 정보
```

## 부트스트랩

```typescript
// packages/opencode/src/cli/bootstrap.ts
export async function bootstrap<T>(
  directory: string,
  cb: () => Promise<T>
): Promise<T>
```

부트스트랩 함수는 다음을 수행한다:
1. `Instance.provide()`로 프로젝트 컨텍스트 생성
2. 내부 초기화 (InstanceBootstrap)
3. 콜백 실행
4. 정리 (finally 블록)

## 슬래시 명령어

TUI/Web 인터페이스 내에서 `/`로 시작하는 슬래시 명령어를 사용할 수 있다.

### 내장 슬래시 명령어

| 명령어 | 설명 |
|--------|------|
| `/compact` | 현재 세션의 메시지를 압축 |
| `/clear` | 세션 초기화 |
| `/model` | 모델 변경 |
| `/agent` | 에이전트 변경 |
| `/session` | 세션 관리 |
| `/help` | 도움말 |

### 커스텀 슬래시 명령어

설정 디렉터리의 `command/` 또는 `commands/` 폴더에 Markdown 파일로 커스텀 명령어를 정의할 수 있다:

```markdown
<!-- ~/.config/opencode/command/deploy.md -->
---
agent: build
model: anthropic/claude-sonnet-4-20250514
---

프로젝트를 배포합니다.
다음 단계를 순서대로 수행하세요:
1. 테스트 실행
2. 빌드
3. 배포 스크립트 실행
```

사용: `/deploy`

## 에러 처리

CLI는 다양한 에러를 처리한다:

```typescript
// NamedError → 구조화된 에러 메시지
// Error → 일반 에러 메시지
// 기타 → 알 수 없는 에러

// 에러 발생 시 로그 파일 경로도 함께 출력
process.exit(1)
```

명시적 `process.exit()`을 호출하여 하위 프로세스가 종료를 차단하지 않도록 한다.

## 참고 문서

- [01. 아키텍처 개요](./01-architecture-overview.md) — CLI의 시스템 내 위치
- [11. 서버 API](./11-server-api.md) — `serve` 명령어의 서버 구조
- [16. 프론트엔드 아키텍처](./16-frontend-architecture.md) — TUI/Web 클라이언트
- [CONTRIBUTING.ko.md](../../../CONTRIBUTING.ko.md) — `bun dev` 사용법
