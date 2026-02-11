# 16. 프론트엔드 아키텍처

이 문서는 OpenCode의 세 가지 클라이언트 인터페이스(TUI, Web, Desktop)의 아키텍처를 설명한다.

## 개요

OpenCode는 하나의 서버에 세 가지 클라이언트가 연결되는 구조이다:

```
┌──────────┐  ┌──────────┐  ┌──────────────┐
│ TUI      │  │ Web UI   │  │ Desktop App  │
│ (터미널)  │  │ (브라우저) │  │ (Tauri)      │
└────┬─────┘  └────┬─────┘  └──────┬───────┘
     │             │               │
     └─────────────┼───────────────┘
                   │
          HTTP / SSE / WebSocket
                   │
           ┌───────▼───────┐
           │ Hono 서버      │
           │ (포트 4096)    │
           └───────────────┘
```

## TUI (Terminal UI)

**소스:** `packages/opencode/src/cli/cmd/tui/`
**프레임워크:** SolidJS + [opentui](https://github.com/sst/opentui)

TUI는 터미널에서 직접 실행되는 인터페이스이다. SolidJS 컴포넌트를 opentui 라이브러리로 터미널에 렌더링한다.

### 컴포넌트 구조

```
tui/
├── app.tsx              # 메인 앱 컴포넌트
├── attach.ts            # 서버 연결
├── event.ts             # 이벤트 처리
├── thread.ts            # 스레드 관리
├── worker.ts            # 백그라운드 작업
├── component/           # 다이얼로그 컴포넌트
│   ├── dialog-agent.tsx     # 에이전트 선택
│   ├── dialog-command.tsx   # 명령어 선택
│   ├── dialog-mcp.tsx       # MCP 관리
│   ├── dialog-model.tsx     # 모델 선택
│   ├── dialog-session-*.tsx # 세션 관리
│   ├── dialog-skill.tsx     # 스킬 선택
│   ├── spinner.tsx          # 로딩 스피너
│   ├── logo.tsx             # 로고
│   └── prompt/              # 프롬프트 입력
│       ├── autocomplete.tsx
│       ├── frecency.tsx     # 빈도 기반 정렬
│       ├── history.tsx      # 입력 히스토리
│       └── stash.tsx        # 임시 저장
├── ui/                  # 기본 UI 요소
│   ├── dialog.tsx
│   ├── dialog-select.tsx
│   ├── toast.tsx
│   └── spinner.ts
├── context/             # SolidJS 컨텍스트
│   ├── args.tsx         # CLI 인자
│   ├── keybind.tsx      # 키바인딩
│   ├── local.tsx        # 로컬 상태
│   ├── prompt.tsx       # 프롬프트 상태
│   ├── route.tsx        # 라우팅
│   ├── sdk.tsx          # SDK 연결
│   ├── sync.tsx         # 동기화
│   └── theme/           # 테마
└── routes/              # 페이지
    ├── home.tsx         # 홈 화면
    └── session/         # 세션 화면
```

### TUI 프로바이더 스택

```
App
└── ThemeProvider        ← 터미널 테마 관리
    └── KeybindProvider  ← 키보드 단축키
        └── RouteProvider
            └── DialogProvider
                └── SDKProvider  ← 서버 SDK 연결
                    └── SyncProvider  ← SSE 이벤트 동기화
                        └── LocalProvider
                            └── Routes (Home / Session)
```

## Web UI

**소스:** `packages/app/src/`
**프레임워크:** SolidJS + Vite + Tailwind CSS

웹 UI는 브라우저에서 실행되며, API 서버와 HTTP/SSE로 통신한다.

### 디렉터리 구조

```
app/src/
├── app.tsx              # 메인 앱 (프로바이더 트리)
├── entry.tsx            # 진입점 (플랫폼별 설정)
├── components/          # UI 컴포넌트
│   ├── prompt-input.tsx     # 프롬프트 입력 (42KB)
│   ├── file-tree.tsx        # 파일 트리
│   ├── terminal.tsx         # 터미널 컴포넌트
│   ├── titlebar.tsx         # 타이틀바
│   ├── dialog-*.tsx         # 각종 다이얼로그
│   ├── settings-*.tsx       # 설정 화면
│   ├── session/             # 세션 관련 컴포넌트
│   └── server/              # 서버 관련 컴포넌트
├── context/             # SolidJS 컨텍스트
│   ├── global-sdk.tsx       # 전역 SDK
│   ├── global-sync.tsx      # 전역 동기화
│   ├── language.tsx         # 국제화
│   ├── layout.tsx           # 레이아웃
│   ├── permission.tsx       # 권한 UI
│   ├── platform.tsx         # 플랫폼 감지
│   ├── sync.tsx             # 이벤트 동기화
│   └── ...
├── pages/               # 페이지
│   ├── home.tsx
│   ├── session.tsx          # 메인 세션 페이지 (56KB)
│   ├── layout.tsx           # 레이아웃 (72KB)
│   └── error.tsx
└── i18n/                # 국제화
    ├── en.ts                # 영어 (40KB)
    ├── ko.ts                # 한국어
    ├── zh.ts                # 중국어 (간체)
    └── ... (16개 언어)
```

### Web 프로바이더 스택

```
MetaProvider
└── Font
    └── ThemeProvider
        └── LanguageProvider        ← 국제화
            └── UiI18nBridge        ← UI 라이브러리 번역 연결
                └── ErrorBoundary
                    └── DialogProvider
                        └── MarkedProvider  ← 마크다운 렌더링
                            └── ServerProvider     ← 서버 연결
                                └── GlobalSDKProvider
                                    └── GlobalSyncProvider
                                        └── Router + Layout
```

## Desktop App

**소스:** `packages/desktop/`
**프레임워크:** Tauri 2.x (Rust 백엔드 + 웹 프론트엔드)

데스크톱 앱은 `packages/app`의 웹 UI를 Tauri 네이티브 창으로 래핑한다.

### Tauri 플러그인

| 플러그인 | 용도 |
|---------|------|
| clipboard-manager | 클립보드 접근 |
| deep-link | URL 스킴 핸들링 |
| dialog | 네이티브 대화상자 |
| opener | 외부 앱 실행 |
| os | OS 정보 |
| notification | 시스템 알림 |
| process | 프로세스 관리 |
| shell | 셸 실행 |
| store | 로컬 저장소 |
| updater | 자동 업데이트 |
| http | HTTP 요청 |
| window-state | 창 상태 저장/복원 |

### 개발 서버

```bash
# 네이티브 창 포함 (Tauri 의존성 필요)
bun run --cwd packages/desktop tauri dev

# 웹 개발 서버만 (http://localhost:1420)
bun run --cwd packages/desktop dev
```

## 국제화 (i18n)

**프레임워크:** `@solid-primitives/i18n`

### 지원 언어

한국어(ko)를 포함하여 16개 언어를 지원한다:
en, ko, zh, zht, ja, de, es, fr, da, pl, ru, ar, no, br, th, bs

### 구현 구조

```typescript
// packages/app/src/context/language.tsx

// 언어 감지 (브라우저)
const browserLang = navigator.languages[0]

// 사전 시스템
const dict = flatten(await import(`../i18n/${locale}.ts`))

// 사용
const { t } = useLanguage()
t("session.title")           // 단순 키
t("command.theme.set", { theme: "dark" })  // 템플릿 보간
```

### 이중 번역 시스템

앱과 UI 라이브러리가 별도의 번역 파일을 가진다:
- `packages/app/src/i18n/` — 앱 번역
- `@opencode-ai/ui/i18n/` — UI 컴포넌트 번역

두 번역은 `LanguageProvider`에서 병합된다.

### 번역 키 일관성

`parity.test.ts` 파일이 모든 언어의 번역 키가 영어 기준과 일치하는지 검증한다.

## 참고 문서

- [01. 아키텍처 개요](./01-architecture-overview.md) — 클라이언트 계층의 위치
- [11. 서버 API](./11-server-api.md) — 클라이언트가 통신하는 서버 API
- [15. CLI와 명령어](./15-cli-and-commands.md) — CLI 명령어
- [CONTRIBUTING.ko.md](../../../CONTRIBUTING.ko.md) — 개발 환경 실행 방법
