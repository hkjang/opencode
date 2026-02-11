# 한영 기술 용어 대조표

이 문서는 OpenCode 한국어 개발자 문서 전체에서 사용하는 기술 용어의 통일 기준을 제공한다.

## 용어 사용 원칙

1. **영어 기술 용어는 원문 유지**: 코드에서 직접 사용하는 식별자, API 이름, 프레임워크 이름 등은 영어 원문 그대로 표기한다.
2. **첫 등장 시 한국어 병기**: 문서에서 처음 등장할 때 "네임스페이스(Namespace)" 형태로 한국어 설명을 병기한다.
3. **이후 등장 시**: 문맥에 따라 한국어 또는 영어 중 자연스러운 쪽을 선택한다.
4. **코드 블록 내**: 식별자는 영문 유지, 주석은 한국어로 작성한다.

## 용어 대조표

### 아키텍처 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| Namespace | 네임스페이스 | TypeScript `export namespace` 패턴 |
| Monorepo | 모노레포 | 단일 저장소에 다수 패키지 |
| Package | 패키지 | npm/Bun 패키지 단위 |
| Module | 모듈 | TypeScript 모듈 |
| Instance | 인스턴스 | 프로젝트 단위 런타임 컨텍스트 |
| Singleton | 싱글턴 | 단일 인스턴스 패턴 |
| Lazy initialization | 지연 초기화 | 최초 접근 시 초기화 |
| Dependency graph | 의존성 그래프 | 패키지 간 의존 관계 |
| Entry point | 진입점 | 프로그램 실행 시작 지점 |
| Middleware | 미들웨어 | 요청 처리 파이프라인의 중간 단계 |

### 핵심 시스템 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| Agent | 에이전트 | AI 역할 정의 (build, plan, general 등) |
| Session | 세션 | 대화 단위 |
| Provider | 프로바이더 | LLM 서비스 공급자 (Anthropic, OpenAI 등) |
| Model | 모델 | LLM 모델 (Claude, GPT 등) |
| Tool | 도구 | 에이전트가 호출하는 기능 단위 |
| Tool Registry | 도구 레지스트리 | 도구 등록 및 관리 모듈 |
| Plugin | 플러그인 | 확장 기능 모듈 |
| Permission | 권한 | 도구 실행 허용/거부 제어 |
| Ruleset | 룰셋 | 권한 규칙 집합 |
| Bus / Event Bus | 이벤트 버스 | 발행-구독 메시지 시스템 |
| Config | 설정 | 환경 구성 값 |
| Storage | 저장소 | 파일 기반 영속 데이터 |
| Snapshot | 스냅샷 | 파일 시스템 상태의 시점별 기록 |

### 세션 처리 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| Processor | 프로세서 | 세션 처리 루프 |
| Stream | 스트림 | LLM 응답의 실시간 데이터 흐름 |
| Message | 메시지 | 세션 내 대화 단위 |
| Part | 파트 | 메시지의 하위 구성 요소 (텍스트, 도구 호출 등) |
| Token | 토큰 | LLM 처리 단위 |
| Compaction | 압축 | 컨텍스트 초과 시 메시지 요약 |
| Fork | 포크 | 기존 세션에서 분기 생성 |

### 서버 및 통신 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| Route | 라우트 | HTTP 엔드포인트 경로 |
| Endpoint | 엔드포인트 | API 접근 지점 |
| Server-Sent Events (SSE) | SSE | 서버→클라이언트 단방향 스트리밍 |
| WebSocket | 웹소켓 | 양방향 실시간 통신 |
| MCP (Model Context Protocol) | MCP | 외부 도구 서버 연동 프로토콜 |
| OAuth | OAuth | 인증 프로토콜 |
| CORS | CORS | 교차 출처 리소스 공유 |
| OpenAPI | OpenAPI | API 명세 표준 |

### 프론트엔드 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| TUI (Terminal UI) | TUI | 터미널 기반 사용자 인터페이스 |
| Web UI | 웹 UI | 브라우저 기반 인터페이스 |
| Desktop App | 데스크톱 앱 | Tauri 기반 네이티브 앱 |
| Component | 컴포넌트 | UI 구성 요소 |
| i18n (Internationalization) | 국제화 | 다국어 지원 |

### 개발 도구 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| Bun | Bun | JavaScript 런타임 및 패키지 관리자 |
| Zod | Zod | TypeScript 스키마 검증 라이브러리 |
| Drizzle | Drizzle | TypeScript ORM |
| Hono | Hono | 경량 웹 프레임워크 |
| SolidJS | SolidJS | 반응형 UI 프레임워크 |
| Tauri | Tauri | 네이티브 앱 프레임워크 (Rust 기반) |
| Yargs | Yargs | CLI 인자 파싱 라이브러리 |
| AI SDK | AI SDK | Vercel AI SDK (LLM 통합 라이브러리) |

### 패턴 및 설계 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| Pub/Sub | 발행-구독 | 이벤트 기반 비동기 통신 패턴 |
| Factory function | 팩토리 함수 | 객체 생성 함수 |
| Higher-order function | 고차 함수 | 함수를 인자로 받거나 반환하는 함수 |
| Type guard | 타입 가드 | TypeScript 타입 좁히기 함수 |
| Discriminated union | 판별 유니온 | 공통 필드로 구분하는 유니온 타입 |
| Schema | 스키마 | 데이터 구조 정의 |
| Wildcard | 와일드카드 | 패턴 매칭에서 모든 값에 대응하는 문자 |
| Fuzzy matching | 퍼지 매칭 | 유사 문자열 검색 |

### 프로세스 용어

| 영어 | 한국어 | 비고 |
|------|--------|------|
| Pull Request (PR) | PR | 코드 병합 요청 |
| Issue | 이슈 | 버그 보고 또는 기능 요청 |
| Commit | 커밋 | 코드 변경 단위 |
| Branch | 브랜치 | 코드 분기 |
| CI/CD | CI/CD | 지속적 통합/배포 |
| Lint | 린트 | 코드 정적 분석 |
| Debug | 디버그 | 오류 추적 및 수정 |
| Breakpoint | 중단점 | 디버거에서 실행을 일시 정지하는 지점 |

## 표기 예시

```markdown
<!-- 첫 등장 -->
OpenCode는 이벤트 버스(Event Bus) 패턴으로 컴포넌트 간 통신을 처리한다.

<!-- 이후 등장 -->
이벤트 버스에 구독하면 세션 변경 알림을 받을 수 있다.
Bus.subscribe()로 특정 이벤트 타입을 구독한다.
```
