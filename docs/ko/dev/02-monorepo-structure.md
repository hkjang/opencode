# 02. 모노레포 구조

이 문서는 OpenCode 모노레포의 패키지 구성, 빌드 시스템, 의존성 그래프를 설명한다.

## 개요

OpenCode는 Bun 워크스페이스(Workspace) 기반 모노레포(Monorepo)로 구성되어 있다. 루트 `package.json`에 워크스페이스가 정의되어 있으며, 기본 브랜치(Branch)는 `dev`이다.

```json
// package.json (루트)
{
  "workspaces": ["packages/*", "packages/console/*", "packages/sdk/js", "packages/slack"]
}
```

## 패키지 구조

```
opencode/
├── packages/
│   ├── opencode/        # 핵심 CLI 및 서버 (메인 패키지)
│   ├── app/             # 웹 UI (SolidJS + Vite)
│   ├── desktop/         # 데스크톱 앱 (Tauri)
│   ├── plugin/          # @opencode-ai/plugin (플러그인 SDK)
│   ├── sdk/js/          # @opencode-ai/sdk (JavaScript SDK)
│   ├── ui/              # 공유 UI 컴포넌트
│   ├── util/            # 공유 유틸리티
│   ├── web/             # 웹 패키지
│   ├── docs/            # 문서
│   ├── script/          # 빌드/생성 스크립트
│   ├── containers/      # Docker 설정
│   ├── extensions/      # 에디터 확장
│   ├── enterprise/      # 엔터프라이즈 기능
│   ├── identity/        # 인증/인가
│   ├── function/        # 서버리스 함수
│   ├── console/         # 콘솔 애플리케이션
│   └── slack/           # Slack 연동
├── CONTRIBUTING.md
├── AGENTS.md
└── package.json         # 루트 워크스페이스 설정
```

## 핵심 패키지 상세

### `packages/opencode` — 메인 패키지

OpenCode의 핵심 비즈니스 로직, CLI, 서버를 포함하는 메인 패키지이다.

**주요 의존성:**
- AI SDK 프로바이더: `@ai-sdk/anthropic`, `@ai-sdk/openai`, `@ai-sdk/google` 등 10+
- 프로토콜: `@modelcontextprotocol/sdk`, `@agentclientprotocol/sdk`
- 내부 패키지: `@opencode-ai/plugin`, `@opencode-ai/sdk`, `@opencode-ai/util`
- 프레임워크: `hono` (HTTP 서버), `yargs` (CLI), `zod` (스키마 검증)
- UI: `@opentui/core`, `@opentui/solid` (TUI 렌더링)

**소스 디렉터리 구조** (`packages/opencode/src/`):

```
src/
├── agent/          # 에이전트 정의 및 시스템 프롬프트
├── auth/           # 인증 (OAuth, API 키)
├── bus/            # 이벤트 버스 (Bus, BusEvent, GlobalBus)
├── cli/            # CLI 명령어 및 TUI
│   ├── cmd/        #   하위 명령어 (serve, web, run, mcp, auth, ...)
│   └── cmd/tui/    #   TUI 컴포넌트 (SolidJS + opentui)
├── command/        # 슬래시 명령어 (/compact, /clear, ...)
├── config/         # 설정 로딩 및 관리
├── env/            # 환경 변수 처리
├── file/           # 파일 시스템 연산
├── flag/           # CLI 플래그
├── format/         # 코드 포매팅
├── global/         # 전역 경로 및 상태
├── id/             # ID 생성 (오름차순/내림차순)
├── ide/            # IDE 연동
├── installation/   # 패키지 설치
├── lsp/            # Language Server Protocol
├── mcp/            # Model Context Protocol
├── patch/          # 코드 패치 적용
├── permission/     # 권한 시스템 (PermissionNext)
├── plugin/         # 플러그인 로딩 및 훅
├── project/        # 프로젝트/인스턴스 관리
├── provider/       # LLM 프로바이더 통합
├── pty/            # 의사 터미널
├── question/       # 사용자 질문 처리
├── scheduler/      # 작업 스케줄링
├── server/         # HTTP 서버 (Hono)
├── session/        # 세션 관리, 메시지, LLM 스트리밍
├── share/          # 세션 공유
├── shell/          # 셸 통합
├── skill/          # 스킬 시스템
├── snapshot/       # 파일 시스템 스냅샷
├── storage/        # 파일 기반 영속 저장소
├── tool/           # 도구 정의 및 레지스트리
├── util/           # 내부 유틸리티
└── worktree/       # Git 워크트리 관리
```

### `packages/app` — 웹 UI

SolidJS와 Vite로 구축된 웹 애플리케이션이다.

**기술 스택:**
- UI: SolidJS, `@kobalte/core` (접근성 컴포넌트)
- 스타일링: Tailwind CSS
- 코드 하이라이팅: Shiki
- 마크다운: marked + marked-shiki
- 국제화: `@solid-primitives/i18n`
- 터미널: `ghostty-web`

**주요 스크립트:**
```bash
bun run --cwd packages/app dev      # 개발 서버 (Vite)
bun run --cwd packages/app build    # 프로덕션 빌드
bun run --cwd packages/app test:e2e # E2E 테스트 (Playwright)
```

### `packages/desktop` — 데스크톱 앱

Tauri 2.x 기반 네이티브 데스크톱 앱으로, `packages/app`을 래핑한다.

**Tauri 플러그인:**
- clipboard-manager, deep-link, dialog, opener, os, notification
- process, shell, store, updater, http, window-state

**주요 스크립트:**
```bash
bun run --cwd packages/desktop tauri dev    # 개발 모드 (네이티브 창)
bun run --cwd packages/desktop tauri build  # 프로덕션 빌드
```

### `packages/plugin` — 플러그인 SDK

외부 플러그인 개발을 위한 `@opencode-ai/plugin` 패키지이다.

**내보내기(Export):**
- `.` → 메인 플러그인 인터페이스
- `./tool` → 도구 정의 헬퍼

**의존성:** `@opencode-ai/sdk`, `zod`

### `packages/sdk/js` — JavaScript SDK

OpenAPI 스펙에서 자동 생성되는 클라이언트 SDK이다.

**내보내기:**
- `.` → 메인 SDK
- `./client` → HTTP 클라이언트
- `./server` → 서버 유틸리티
- `./v2` → v2 API 클라이언트
- `./v2/client` → v2 HTTP 클라이언트

**재생성:** SDK나 API를 변경한 후 실행:
```bash
./script/generate.ts
```

## 의존성 그래프

```
opencode ──┬── plugin ──── sdk
           ├── sdk
           └── util

app ──┬── sdk
      ├── ui
      └── util

desktop ──┬── app
          └── ui
```

핵심 의존 방향:
- `opencode` → `plugin`, `sdk`, `util` (핵심 로직이 내부 패키지 사용)
- `app` → `sdk`, `ui`, `util` (웹 UI가 SDK로 서버와 통신)
- `desktop` → `app`, `ui` (Tauri가 웹 앱을 래핑)
- `plugin` → `sdk` (플러그인이 SDK로 시스템과 상호작용)
- `sdk` → 외부 의존성 없음 (순수 생성 코드)

## 빌드 시스템

### 개발 모드

```bash
# 루트에서 실행
bun install                          # 의존성 설치
bun dev                              # TUI 개발 서버 (기본)
bun dev serve                        # API 서버만
bun dev web                          # 서버 + 웹 UI
bun run --cwd packages/app dev       # 웹 UI만 (별도 서버 필요)
bun run --cwd packages/desktop tauri dev  # 데스크톱 앱
```

### 프로덕션 빌드

```bash
# 독립 실행 파일 빌드
./packages/opencode/script/build.ts --single

# 데스크톱 앱 빌드
bun run --cwd packages/desktop tauri build
```

### 타입 체크

```bash
bun turbo typecheck    # 모든 패키지 타입 체크 (Turborepo)
```

### SDK 재생성

API 스펙이 변경된 경우:
```bash
./script/generate.ts
```

SDK 단독 재생성:
```bash
./packages/sdk/js/script/build.ts
```

## 참고 문서

- [01. 아키텍처 개요](./01-architecture-overview.md) — 전체 시스템 설계
- [03. 핵심 패턴](./03-core-patterns.md) — 코드베이스 공통 패턴
- [CONTRIBUTING.ko.md](../../../CONTRIBUTING.ko.md) — 개발 환경 설정
