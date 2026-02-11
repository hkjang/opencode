# OpenCode 개발자 문서 (한국어)

OpenCode 코드베이스를 이해하고 수정하기 위한 개발자 문서입니다. 사용자 문서는 [공식 문서 사이트](https://opencode.ai/docs)를 참고하세요.

## 시작하기

| 문서 | 설명 |
|------|------|
| [CONTRIBUTING.ko.md](../../../CONTRIBUTING.ko.md) | 기여 가이드 — 개발 환경, PR 규칙, 디버깅 |
| [AGENTS.ko.md](../../../AGENTS.ko.md) | 코드 스타일 가이드 |
| [용어 사전](./20-glossary.md) | 한영 기술 용어 대조표 |

## 아키텍처 기반

| 문서 | 설명 |
|------|------|
| [01. 아키텍처 개요](./01-architecture-overview.md) | 시스템 전체 구조, 요청 흐름, 계층 구조 |
| [02. 모노레포 구조](./02-monorepo-structure.md) | 패키지 구성, 빌드 시스템, 의존성 그래프 |
| [03. 핵심 패턴](./03-core-patterns.md) | Namespace, Zod, Instance.state(), BusEvent 등 |

## 핵심 시스템

| 문서 | 설명 |
|------|------|
| [04. 에이전트 시스템](./04-agent-system.md) | build/plan/general 에이전트, 시스템 프롬프트 |
| [05. 세션 생명주기](./05-session-lifecycle.md) | 세션 생성 → 메시지 처리 → LLM 스트리밍 → 도구 실행 |
| [06. 프로바이더 시스템](./06-provider-system.md) | 20+ LLM 프로바이더 통합 |
| [07. 도구 시스템](./07-tool-system.md) | 40+ 내장 도구, Tool.define(), ToolRegistry |
| [08. 이벤트 버스](./08-bus-event-system.md) | Bus/GlobalBus, BusEvent.define(), SSE 연동 |
| [09. 권한 시스템](./09-permission-system.md) | PermissionNext, allow/deny/ask, Ruleset |
| [10. 설정 시스템](./10-config-system.md) | 설정 로딩 우선순위, Config.get(), JSONC |

## 인프라

| 문서 | 설명 |
|------|------|
| [11. 서버 API](./11-server-api.md) | Hono HTTP 서버, 라우트 구조, OpenAPI |
| [12. 플러그인 시스템](./12-plugin-system.md) | 플러그인 인터페이스, 내부/외부 플러그인 |
| [13. MCP 통합](./13-mcp-integration.md) | MCP 클라이언트, 도구 변환, OAuth |
| [14. 저장소와 상태](./14-storage-and-state.md) | Storage, Snapshot, Instance.state() |
| [15. CLI와 명령어](./15-cli-and-commands.md) | Yargs CLI, 슬래시 명령어, 부트스트랩 |
| [16. 프론트엔드 아키텍처](./16-frontend-architecture.md) | TUI/Web/Desktop 클라이언트, i18n |

## 실전 가이드

| 문서 | 설명 |
|------|------|
| [17. 테스트와 디버깅](./17-testing-and-debugging.md) | 테스트 철학, Bun 디버거, VSCode 설정 |
| [18. 프로바이더 추가하기](./18-adding-a-provider.md) | 새 LLM 프로바이더 추가 가이드 |
| [19. 도구 추가하기](./19-adding-a-tool.md) | 새 도구 추가 가이드 |

## 심화 모듈 문서

| 문서 | 설명 |
|------|------|
| [21. 인증과 세션 공유](./21-auth-and-sharing.md) | Auth 모듈, OAuth/API 키, 세션 공유 시스템 |
| [22. 명령어와 스킬](./22-command-and-skill.md) | 슬래시 명령어, 스킬 검색, 원격 스킬 |
| [23. 세션 내부 구현](./23-session-internals.md) | 프롬프트 오케스트레이션, 압축, 재시도, 되돌리기 |
| [24. 파일 시스템과 검색](./24-file-and-search.md) | Ripgrep, 파일 감시, 무시 패턴, 수정 시간 추적 |
| [25. 도구 구현 상세](./25-tool-implementation-details.md) | 19개 내장 도구의 파라미터, 권한, 핵심 로직 |
| [26. LSP와 포매팅](./26-lsp-and-formatting.md) | 30+ LSP 서버, 30+ 포매터, 자동 감지 |
| [27. 인프라 모듈](./27-infrastructure-modules.md) | Global, Env, ID, Shell, 설치, 스케줄러, Pty, Worktree |
| [28. ACP 프로토콜](./28-acp-protocol.md) | Agent Client Protocol, IDE 연동 |
| [29. 패치 시스템](./29-patch-system.md) | 커스텀 패치 형식, 4단계 줄 매칭, 역순 적용 |
| [30. 질문과 사용자 상호작용](./30-question-and-interaction.md) | Question 시스템, 권한 다이얼로그, UI 연동 |

## 문서 작성 규칙

- 영어 기술 용어는 원문 유지, 첫 등장 시 한국어 병기 (예: "네임스페이스(Namespace)")
- 코드 내 주석은 한국어, 식별자는 영문 유지
- 용어 통일은 [용어 사전](./20-glossary.md) 기준
