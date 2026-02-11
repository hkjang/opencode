# OpenCode 기여 가이드

[English version](./CONTRIBUTING.md)

OpenCode에 기여해 주셔서 감사합니다. 아래에 병합 가능성이 높은 기여 유형을 안내합니다.

- 버그 수정
- LSP / 포매터 추가
- LLM 성능 개선
- 새 프로바이더(Provider) 지원
- 환경별 이슈 해결
- 누락된 표준 동작 추가
- 문서 개선

단, **UI 변경이나 핵심 기능 추가**는 구현 전에 반드시 코어 팀의 디자인 리뷰를 거쳐야 합니다.

PR이 수용될지 확신이 없다면, 메인테이너에게 문의하거나 다음 라벨이 붙은 이슈를 참고하세요:

- [`help wanted`](https://github.com/anomalyco/opencode/issues?q=is%3Aissue%20state%3Aopen%20label%3Ahelp-wanted)
- [`good first issue`](https://github.com/anomalyco/opencode/issues?q=is%3Aissue%20state%3Aopen%20label%3A%22good%20first%20issue%22)
- [`bug`](https://github.com/anomalyco/opencode/issues?q=is%3Aissue%20state%3Aopen%20label%3Abug)
- [`perf`](https://github.com/anomalyco/opencode/issues?q=is%3Aopen%20is%3Aissue%20label%3A%22perf%22)

> [!NOTE]
> 이 가이드라인을 무시한 PR은 닫힐 수 있습니다.

이슈를 작업하고 싶다면 댓글을 남겨 주세요. 이미 진행 중이 아닌 경우 메인테이너가 할당해 줄 수 있습니다.

## 개발 환경 설정

- 요구 사항: **Bun 1.3 이상**
- 저장소 루트에서 의존성 설치 및 개발 서버 실행:

```bash
bun install
bun dev
```

### 다른 디렉터리에서 실행하기

기본적으로 `bun dev`는 `packages/opencode` 디렉터리를 대상으로 OpenCode를 실행합니다. 다른 디렉터리나 저장소를 대상으로 실행하려면:

```bash
bun dev <디렉터리>
```

opencode 저장소 루트에서 실행하려면:

```bash
bun dev .
```

### 로컬 빌드 ("localcode")

독립 실행 파일을 컴파일하려면:

```bash
./packages/opencode/script/build.ts --single
```

실행:

```bash
./packages/opencode/dist/opencode-<플랫폼>/bin/opencode
```

`<플랫폼>`은 `darwin-arm64`, `linux-x64` 등으로 교체하세요.

### 핵심 패키지 구조

- `packages/opencode`: OpenCode 핵심 비즈니스 로직 및 서버
- `packages/opencode/src/cli/cmd/tui/`: TUI 코드 (SolidJS + [opentui](https://github.com/sst/opentui))
- `packages/app`: 공유 웹 UI 컴포넌트(SolidJS)
- `packages/desktop`: 네이티브 데스크톱 앱 (Tauri, `packages/app`을 래핑)
- `packages/plugin`: `@opencode-ai/plugin` 소스

> 전체 패키지 구조는 [모노레포 구조 문서](./docs/ko/dev/02-monorepo-structure.md)를 참고하세요.

### `bun dev`와 `opencode` 명령어의 관계

개발 시 `bun dev`는 빌드된 `opencode` 명령어와 동일한 CLI를 실행합니다:

```bash
# 개발 환경 (프로젝트 루트)
bun dev --help           # 사용 가능한 명령어 표시
bun dev serve            # 헤드리스 API 서버 시작
bun dev web              # 서버 + 웹 인터페이스 시작
bun dev <디렉터리>       # 특정 디렉터리에서 TUI 시작

# 프로덕션
opencode --help          # 사용 가능한 명령어 표시
opencode serve           # 헤드리스 API 서버 시작
opencode web             # 서버 + 웹 인터페이스 시작
opencode <디렉터리>      # 특정 디렉터리에서 TUI 시작
```

### API 서버 실행

```bash
bun dev serve
```

기본 포트는 4096입니다. 다른 포트를 지정하려면:

```bash
bun dev serve --port 8080
```

### 웹 앱 실행

UI 변경 사항을 테스트하려면:

1. **먼저 OpenCode 서버를 시작**합니다 (위의 [API 서버 실행](#api-서버-실행) 참조)
2. **그 다음 웹 앱을 실행**합니다:

```bash
bun run --cwd packages/app dev
```

http://localhost:5173 에서 로컬 개발 서버가 시작됩니다. 대부분의 UI 변경은 여기서 테스트할 수 있지만, 전체 기능을 사용하려면 서버가 실행 중이어야 합니다.

### 데스크톱 앱 실행

데스크톱 앱은 웹 UI를 감싸는 네이티브 Tauri 앱입니다.

```bash
bun run --cwd packages/desktop tauri dev
```

http://localhost:1420 에서 웹 개발 서버가 시작되고 네이티브 창이 열립니다.

웹 개발 서버만 실행하려면 (네이티브 셸 없이):

```bash
bun run --cwd packages/desktop dev
```

프로덕션 빌드:

```bash
bun run --cwd packages/desktop tauri build
```

> [!NOTE]
> 데스크톱 앱 실행에는 Tauri 의존성(Rust 툴체인, 플랫폼별 라이브러리)이 필요합니다. [Tauri 사전 요구 사항](https://v2.tauri.app/start/prerequisites/)을 참고하세요.

> [!NOTE]
> API나 SDK를 변경한 경우 (`packages/opencode/src/server/server.ts` 등), `./script/generate.ts`를 실행하여 SDK 및 관련 파일을 재생성하세요.

코드 스타일은 [스타일 가이드](./AGENTS.ko.md)를 따라 주세요.

### 디버거 설정

Bun 디버깅은 아직 안정화되지 않은 부분이 있습니다. 다음 가이드를 참고하세요.

가장 안정적인 방법은 터미널에서 `bun run --inspect=<url> dev ...`로 실행한 뒤, 해당 URL로 디버거를 연결하는 것입니다.

주의사항:

- TUI를 실행하면서 서버 코드에 중단점(Breakpoint)을 사용하려면 `bun dev` 대신 `bun dev spawn`을 사용해야 할 수 있습니다. `bun dev`는 서버를 워커 스레드에서 실행하므로 중단점이 동작하지 않을 수 있기 때문입니다.
- `spawn`이 동작하지 않으면 서버를 별도로 디버그할 수 있습니다:
  - 서버 디버그: `bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096`,
    이후 TUI 연결: `opencode attach http://localhost:4096`
  - TUI 디버그: `bun run --inspect=ws://localhost:6499/ --cwd packages/opencode --conditions=browser ./src/index.ts`

기타 팁:

- 워크플로에 따라 `--inspect` 대신 `--inspect-wait`이나 `--inspect-brk`를 사용하는 것이 편리할 수 있습니다.
- 매번 `--inspect=ws://localhost:6499/`를 지정하는 대신 `export BUN_OPTIONS=--inspect=ws://localhost:6499/`를 설정해 두면 편리합니다.

#### VSCode 설정

VSCode를 사용한다면 예제 설정 파일을 참고하세요:
- [.vscode/settings.example.json](.vscode/settings.example.json)
- [.vscode/launch.example.json](.vscode/launch.example.json)

다음 디버그 방법은 문제가 있을 수 있습니다:

- `"request": "launch"` 디버그 설정은 중단점이 잘못 매핑될 수 있습니다.
- VSCode의 `JavaScript Debug Terminal`도 같은 문제가 발생할 수 있습니다.

다만 환경에 따라 동작할 수도 있으므로 시도해 보셔도 좋습니다.

## Pull Request 규칙

### 이슈 우선 정책

**모든 PR은 기존 이슈를 참조해야 합니다.** PR을 열기 전에 버그나 기능을 설명하는 이슈를 먼저 만드세요. 이를 통해 메인테이너가 분류할 수 있고 중복 작업을 방지할 수 있습니다. 연결된 이슈 없는 PR은 리뷰 없이 닫힐 수 있습니다.

- PR 설명에 `Fixes #123` 또는 `Closes #123`을 사용하여 이슈를 연결하세요.
- 작은 수정의 경우, 메인테이너가 문제를 이해할 수 있을 정도의 간단한 이슈면 충분합니다.

### 일반 요구 사항

- PR은 작고 집중된 범위로 유지하세요.
- 이슈와 그 해결 방법을 설명하세요.
- 새 기능을 추가하기 전에, 코드베이스에 이미 동일한 기능이 없는지 확인하세요.

### UI 변경

UI 변경이 포함된 PR에는 **변경 전후의 스크린샷이나 동영상**을 첨부하세요. 메인테이너가 더 빠르게 리뷰할 수 있습니다.

### 로직 변경

비-UI 변경(버그 수정, 새 기능, 리팩토링)의 경우, **동작 검증 방법**을 설명하세요:

- 무엇을 테스트했는지?
- 리뷰어가 어떻게 재현/확인할 수 있는지?

### AI 생성 텍스트 금지

길고 AI가 생성한 PR 설명이나 이슈는 수용되지 않으며 무시될 수 있습니다. 메인테이너의 시간을 존중하세요:

- 짧고 집중된 설명을 작성하세요.
- 무엇이 변경되었고 왜 변경했는지 본인의 말로 설명하세요.
- 간략히 설명할 수 없다면, PR이 너무 클 수 있습니다.

### PR 제목

PR 제목은 Conventional Commit 표준을 따릅니다:

- `feat:` 새 기능
- `fix:` 버그 수정
- `docs:` 문서 변경
- `chore:` 유지보수, 의존성 업데이트 등
- `refactor:` 동작 변경 없는 리팩토링
- `test:` 테스트 추가/수정

선택적으로 영향받는 패키지를 스코프로 포함할 수 있습니다:

- `feat(app):` app 패키지의 기능
- `fix(desktop):` desktop 패키지의 버그 수정
- `chore(opencode):` opencode 패키지의 유지보수

예시:

- `docs: update contributing guidelines`
- `fix: resolve crash on startup`
- `feat: add dark mode support`
- `feat(app): add dark mode support`
- `fix(desktop): resolve crash on startup`
- `chore: bump dependency versions`

### 코드 스타일

엄격하게 강제되진 않지만, 일반적인 가이드라인입니다:

- **함수**: 재사용이나 합성의 이점이 명확하지 않으면 하나의 함수에 로직을 유지하세요.
- **구조 분해**: 불필요한 변수 구조 분해를 하지 마세요.
- **제어 흐름**: `else` 문을 피하세요.
- **에러 처리**: 가능하면 `try`/`catch` 대신 `.catch(...)`를 사용하세요.
- **타입**: 정밀한 타입을 사용하고 `any`를 피하세요.
- **변수**: 불변 패턴을 고수하고 `let`을 피하세요.
- **명명**: 설명이 충분하다면 간결한 단일 단어 식별자를 선택하세요.
- **런타임 API**: `Bun.file()` 등 Bun 헬퍼를 적극 사용하세요.

> 더 자세한 스타일 가이드는 [AGENTS.ko.md](./AGENTS.ko.md)를 참고하세요.

## 기능 제안

새로운 기능은 먼저 설계 논의부터 시작하세요. 문제, 제안하는 접근법(선택 사항), 그리고 OpenCode에 왜 포함되어야 하는지를 설명하는 이슈를 열어 주세요. 코어 팀이 진행 여부를 결정하므로, 기능 PR을 바로 열지 마시고 승인을 기다려 주세요.

## 신뢰 및 보증(Vouch) 시스템

이 프로젝트는 [vouch](https://github.com/mitchellh/vouch)를 사용하여 기여자 신뢰를 관리합니다. 보증 목록은 [`.github/VOUCHED.td`](.github/VOUCHED.td)에서 관리됩니다.

### 동작 방식

- **보증된 사용자(Vouched)**: 명시적으로 신뢰된 기여자입니다.
- **차단된 사용자(Denounced)**: 명시적으로 차단된 사용자입니다. 차단된 사용자의 이슈와 PR은 자동으로 닫힙니다. 차단된 경우 [Discord](https://opencode.ai/discord)에서 메인테이너에게 해제를 요청할 수 있습니다.
- **그 외**: 보증 없이도 정상적으로 참여할 수 있습니다.

### 메인테이너 가이드

쓰기 권한이 있는 협력자는 이슈에 댓글로 보증 목록을 관리할 수 있습니다:

- `vouch` — 이슈 작성자를 보증
- `vouch @username` — 특정 사용자를 보증
- `denounce` — 이슈 작성자를 차단
- `denounce @username` — 특정 사용자를 차단
- `denounce @username <사유>` — 사유와 함께 차단
- `unvouch` / `unvouch @username` — 목록에서 제거

변경 사항은 `.github/VOUCHED.td`에 자동으로 커밋됩니다.

### 차단 정책

차단은 저품질 AI 생성 기여를 반복적으로 제출하거나, 스팸을 보내거나, 악의적으로 행동하는 사용자에게 적용됩니다. 의견 불일치나 실수에는 사용되지 않습니다.

## 이슈 요구 사항

모든 이슈는 반드시 다음 템플릿 중 하나를 사용해야 합니다:

- **Bug report** — 버그 보고 (설명 필수)
- **Feature request** — 기능 제안 (확인 체크박스 및 설명 필수)
- **Question** — 질문 (질문 내용 필수)

빈 이슈는 허용되지 않습니다. 새 이슈가 열리면 자동화 검사가 템플릿 준수 여부와 기여 가이드라인 충족 여부를 확인합니다. 요구 사항을 충족하지 않으면 수정 사항을 안내하는 댓글이 달리며, **2시간** 이내에 수정해야 합니다. 이후에는 자동으로 닫힙니다.

다음 이유로 이슈가 표시될 수 있습니다:

- 템플릿 미사용
- 필수 필드가 비어 있거나 자리 표시자 텍스트
- AI 생성 장문
- 의미 있는 내용 부재

잘못 표시되었다고 판단되면 메인테이너에게 알려 주세요.
