# 27. 인프라 모듈

이 문서는 OpenCode의 기반 인프라 모듈들을 설명한다. 전역 경로, 환경 변수, ID 생성, 셸, 설치 관리, 스케줄러, 의사 터미널, Git 워크트리를 다룬다.

## Global: 전역 경로

**소스:** `packages/opencode/src/global/index.ts`

XDG Base Directory 표준을 따르는 전역 경로 관리이다.

```typescript
export namespace Global {
  export const Path = {
    home:   string  // 사용자 홈 디렉터리
    data:   string  // ~/.local/share/opencode (XDG_DATA_HOME)
    bin:    string  // 바이너리 디렉터리
    log:    string  // 로그 디렉터리
    cache:  string  // ~/.cache/opencode (XDG_CACHE_HOME)
    config: string  // ~/.config/opencode (XDG_CONFIG_HOME)
    state:  string  // ~/.local/state/opencode (XDG_STATE_HOME)
  }
}
```

**캐시 버전 관리:** `Global.Path.cache/version` 파일에 버전 번호(현재 "21")를 저장한다. 버전 불일치 시 캐시를 자동 삭제하여 호환성 문제를 방지한다.

**테스트 격리:** `OPENCODE_TEST_HOME` 환경 변수로 모든 경로를 테스트 디렉터리로 리디렉션할 수 있다.

## Env: 환경 변수 격리

**소스:** `packages/opencode/src/env/index.ts`

인스턴스(Instance)별로 격리된 환경 변수를 관리한다.

```typescript
export namespace Env {
  export function get(key: string): string | undefined
  export function all(): Record<string, string>
  export function set(key: string, value: string): void
  export function remove(key: string): void
}
```

**동작 원리:**
- `Instance.state()`를 사용하여 `process.env`의 얕은 복사본을 인스턴스별로 관리
- 각 프로젝트 인스턴스가 독립적인 환경 변수를 가짐
- `process.env`를 직접 수정하지 않으므로 테스트 병렬화가 안전

## Identifier: ID 생성

**소스:** `packages/opencode/src/id/id.ts`

정렬 가능한 고유 ID를 생성한다.

```typescript
export namespace Identifier {
  // Zod 스키마 (접두어 검증)
  export function schema(prefix: string): z.ZodString

  // 단조 증가 ID (시간순 정렬)
  export function ascending(prefix: string, given?: string): string

  // 단조 감소 ID (역시간순 정렬)
  export function descending(prefix: string, given?: string): string

  // ID에서 타임스탬프 추출
  export function timestamp(id: string): number
}
```

### ID 형식

```
{접두어}_{12자리_타임스탬프_hex}_{10자리_랜덤_base62}

예: ses_018f9a3b4c00_aB3dE7fG9h
    msg_018f9a3b4c01_xY2mN8pQ5r
```

**접두어 매핑:**

| 엔티티 | 접두어 |
|--------|--------|
| Session | `ses` |
| Message | `msg` |
| Part | `prt` |
| Permission | `per` |
| Question | `que` |
| User | `usr` |
| Pty | `pty` |
| Tool | `tool` |

**단조 생성:** 같은 밀리초 내에서 호출되면 카운터를 증가시켜 순서를 보장한다.

## Shell: 셸 유틸리티

**소스:** `packages/opencode/src/shell/shell.ts`

```typescript
export namespace Shell {
  // 프로세스 트리 종료
  export function killTree(proc: ChildProcess, options?: {
    exited?: boolean
  }): Promise<void>

  // 선호 셸 (SHELL 환경 변수 또는 플랫폼 기본)
  export const preferred: string

  // 허용 셸 (fish/nu 제외)
  export const acceptable: string
}
```

**플랫폼별 종료:**
- **Windows**: `taskkill /f /t /pid {pid}` (프로세스 트리 강제 종료)
- **Unix**: `SIGTERM` 전송 후 200ms 대기, 미종료 시 `SIGKILL`

**플랫폼별 기본 셸:**
- Windows: Git Bash (있으면) 또는 `cmd.exe`
- macOS: `/bin/zsh`
- Linux: `/bin/bash` 또는 `/bin/sh`

## Installation: 설치 및 업그레이드

**소스:** `packages/opencode/src/installation/index.ts`

```typescript
export namespace Installation {
  // 현재 버전 + 최신 버전
  export function info(): Promise<{ version: string, latest: string }>

  // 설치 방식 감지
  export function method(): "curl" | "npm" | "yarn" | "pnpm" | "bun" | "brew" | "scoop" | "choco"

  // 업그레이드 실행
  export async function upgrade(method: string, target?: string): Promise<void>

  // 최신 버전 조회
  export async function latest(installMethod?: string): Promise<string>

  // 프리뷰/로컬 빌드 확인
  export function isPreview(): boolean
  export function isLocal(): boolean
}
```

**설치 방식별 업그레이드:**

| 방식 | 명령 |
|------|------|
| npm | `npm install -g opencode` |
| bun | `bun install -g opencode` |
| brew | `brew upgrade opencode` |
| scoop | `scoop update opencode` |
| choco | `choco upgrade opencode` |
| curl | curl 스크립트 재실행 |

**User-Agent 형식:** `opencode/{channel}/{version}/{client}`

## Scheduler: 작업 스케줄러

**소스:** `packages/opencode/src/scheduler/index.ts`

```typescript
export namespace Scheduler {
  export function register(task: {
    id: string
    interval: number          // 밀리초
    run: () => Promise<void>
    scope?: "instance" | "global"
  }): void
}
```

- `"instance"` 범위: 인스턴스 해제 시 타이머 정리
- `"global"` 범위: 앱 전체 수명 동안 실행
- 에러는 로깅하고 타이머는 계속 실행

**사용 예:** 스냅샷의 `git gc` (매시간), 캐시 정리 등.

## Pty: 의사 터미널

**소스:** `packages/opencode/src/pty/index.ts`

웹소켓(WebSocket) 기반 의사 터미널(Pseudo Terminal) 세션 관리이다.

```typescript
export namespace Pty {
  export async function create(input?: {
    command?: string
    args?: string[]
    cwd?: string
    title?: string
    env?: Record<string, string>
  }): Promise<Info>

  export function write(id: string, data: string): void
  export function resize(id: string, cols: number, rows: number): void
  export function remove(id: string): Promise<void>
  export function connect(id: string, ws: WebSocket): void
  export function list(): Info[]
  export function get(id: string): Info | undefined
}
```

### 웹소켓 스트리밍

```
클라이언트 (WebSocket)
    │
    ├── → 입력 데이터 (키 입력)
    │
    └── ← 출력 스트림
         ├── 일반 데이터: raw 터미널 출력
         └── 제어 프레임: 0x00 + JSON({ cursor: { row, col } })
```

**버퍼 관리:**
- 2MB 히스토리 버퍼 (FIFO 오버플로)
- 64KB 단위 스트리밍
- 클라이언트 연결 시 히스토리 재생

### 셸 감지

플랫폼별 기본 셸을 자동 감지하고, Windows에서는 UTF-8 환경을 설정한다.

### 이벤트

```
pty.created   — 새 PTY 생성
pty.updated   — 제목/크기 변경
pty.exited    — 프로세스 종료
pty.deleted   — PTY 삭제
```

## Worktree: Git 워크트리

**소스:** `packages/opencode/src/worktree/index.ts`

Git 워크트리를 활용한 격리된 작업 공간 관리이다.

```typescript
export namespace Worktree {
  export async function create(input: {
    branch?: string
    name?: string
    startCommands?: string[]
  }): Promise<Info>

  export async function remove(input: {
    name: string
  }): Promise<void>

  export async function reset(input: {
    name: string
  }): Promise<void>

  export const Event = {
    Ready: BusEvent.define("worktree.ready", ...),
    Failed: BusEvent.define("worktree.failed", ...),
  }
}
```

### 워크트리 구조

```
~/.local/share/opencode/worktree/{projectID}/
├── brave-eagle-xyz/    ← 자동 생성된 이름
│   └── (프로젝트 파일)
└── calm-tiger-abc/
    └── (프로젝트 파일)
```

**브랜치 이름:** `opencode/{워크트리이름}`

**이름 생성:** 형용사 + 명사 + 랜덤 접미어 (예: `brave-eagle-xyz`)

### 생성 과정

```
1. 고유 이름 생성
2. git worktree add --detach
3. git checkout -b opencode/{name}
4. 서브모듈 초기화 (git submodule init/update)
5. 프로젝트 시작 명령 실행 (bun install 등)
6. Worktree.Event.Ready 이벤트 발행
```

### 에러 타입

```typescript
NotGitError                  // Git 저장소가 아님
NameGenerationFailedError    // 이름 생성 실패
CreateFailedError            // 워크트리 생성 실패
RemoveFailedError            // 워크트리 제거 실패
ResetFailedError             // 워크트리 초기화 실패
```

## 참고 문서

- [02. 모노레포 구조](./02-monorepo-structure.md) — 패키지 빌드 시스템
- [03. 핵심 패턴](./03-core-patterns.md) — Instance.state() 패턴
- [14. 저장소와 상태](./14-storage-and-state.md) — Storage 경로
- [15. CLI와 명령어](./15-cli-and-commands.md) — 업그레이드 명령
