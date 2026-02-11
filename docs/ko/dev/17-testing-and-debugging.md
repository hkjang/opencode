# 17. 테스트와 디버깅

이 문서는 OpenCode의 테스트 철학, 테스트 실행 방법, 디버깅 기법을 설명한다.

## 테스트 철학

OpenCode의 테스트 원칙 ([AGENTS.ko.md](../../../AGENTS.ko.md) 참조):

1. **모의 객체(Mock) 최소화**: 가능한 한 실제 구현을 테스트한다
2. **로직 중복 금지**: 테스트에 비즈니스 로직을 복제하지 않는다
3. **실제 동작 검증**: 단위 테스트보다 통합 테스트를 선호한다

## 테스트 실행

### 패키지별 테스트

```bash
# opencode 패키지 테스트
bun test --cwd packages/opencode

# app 패키지 단위 테스트
bun run --cwd packages/app test:unit

# app E2E 테스트 (Playwright)
bun run --cwd packages/app test:e2e
```

### 개별 테스트 파일 실행

```bash
bun test packages/opencode/src/config/config.test.ts
```

### 타입 체크

```bash
bun turbo typecheck    # 모든 패키지 타입 체크
```

## 디버거 설정

### Bun 디버거 기본

Bun 디버깅은 아직 안정화 중이므로 몇 가지 주의가 필요하다.

**가장 안정적인 방법:** 터미널에서 `--inspect` 플래그로 실행한 뒤 디버거를 연결한다.

```bash
# 서버 디버깅
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096

# TUI 디버깅
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode --conditions=browser ./src/index.ts

# 서버와 TUI 분리 디버깅
# 1. 서버 시작 (디버거 연결)
bun run --inspect=ws://localhost:6499/ --cwd packages/opencode ./src/index.ts serve --port 4096
# 2. TUI 연결
opencode attach http://localhost:4096
```

### TUI + 서버 함께 디버깅

`bun dev`는 서버를 워커 스레드에서 실행하므로 중단점(Breakpoint)이 동작하지 않을 수 있다. 대안:

```bash
# spawn 모드로 실행 (서버를 별도 프로세스로 시작)
bun dev spawn
```

### 유용한 팁

```bash
# 환경 변수로 inspect 옵션 고정
export BUN_OPTIONS=--inspect=ws://localhost:6499/

# 시작 시 대기 (디버거가 연결될 때까지)
bun run --inspect-wait=ws://localhost:6499/ ...

# 첫 줄에서 중단
bun run --inspect-brk=ws://localhost:6499/ ...
```

### VSCode 설정

예제 설정 파일 참고:
- `.vscode/settings.example.json`
- `.vscode/launch.example.json`

**주의사항:**
- `"request": "launch"` 디버그 설정은 중단점이 잘못 매핑될 수 있음
- `JavaScript Debug Terminal`도 같은 문제가 발생할 수 있음
- 가장 안정적인 방법은 `"request": "attach"`로 이미 실행 중인 프로세스에 연결하는 것

```jsonc
// .vscode/launch.json 예시
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to Bun",
      "address": "ws://localhost:6499/",
      "localRoot": "${workspaceFolder}",
      "remoteRoot": "${workspaceFolder}"
    }
  ]
}
```

## 디버깅 유틸리티

### debug 명령어

```bash
opencode debug config    # 현재 설정 출력 (병합 결과)
opencode debug agent     # 에이전트 설정 확인
opencode debug file      # 파일 시스템 정보
opencode debug lsp       # LSP 서버 상태
opencode debug snapshot  # 스냅샷 정보
opencode debug ripgrep   # Ripgrep 동작 확인
opencode debug skill     # 스킬 정보
```

### 로그 출력

```bash
# stderr로 로그 출력
opencode --print-logs

# 로그 레벨 설정
opencode --log-level DEBUG   # DEBUG, INFO, WARN, ERROR
```

개발 환경에서는 기본 로그 레벨이 `DEBUG`이고, 프로덕션에서는 `INFO`이다.

### 환경 변수

디버깅에 유용한 환경 변수:

| 변수 | 설명 |
|------|------|
| `OPENCODE_EXPERIMENTAL_LSP_TOOL` | LSP 도구 활성화 |
| `OPENCODE_ENABLE_EXA` | 웹/코드 검색 활성화 |
| `OPENCODE_DISABLE_DEFAULT_PLUGINS` | 기본 플러그인 비활성화 |
| `OPENCODE_CONFIG` | 커스텀 설정 파일 경로 |
| `OPENCODE_CONFIG_CONTENT` | 인라인 설정 JSON |
| `OPENCODE_SERVER_PASSWORD` | 서버 인증 비밀번호 |

## E2E 테스트 (Web)

`packages/app`의 E2E 테스트는 Playwright를 사용한다.

```bash
# E2E 테스트 실행
bun run --cwd packages/app test:e2e

# 로컬 E2E 테스트 (헤드풀 모드)
bun run --cwd packages/app test:e2e:local

# E2E UI 모드 (시각적 디버깅)
bun run --cwd packages/app test:e2e:ui

# 테스트 리포트 확인
bun run --cwd packages/app test:e2e:report
```

## 번역 키 검증

국제화(i18n) 번역 키의 일관성을 검증한다:

```bash
bun test packages/app/src/i18n/parity.test.ts
```

영어 기준 파일의 모든 키가 다른 언어에도 존재하는지 확인한다.

## 참고 문서

- [CONTRIBUTING.ko.md](../../../CONTRIBUTING.ko.md) — 개발 환경 설정 및 디버거 설정
- [AGENTS.ko.md](../../../AGENTS.ko.md) — 테스트 원칙
- [02. 모노레포 구조](./02-monorepo-structure.md) — 패키지별 빌드/테스트 명령어
