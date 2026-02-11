# 07. 도구 시스템

이 문서는 OpenCode의 도구(Tool) 시스템을 설명한다. 에이전트가 파일 시스템, 셸, 웹 검색 등을 수행할 수 있게 하는 실행 단위이다.

**소스 파일:**
- `packages/opencode/src/tool/tool.ts` — 도구 인터페이스 및 팩토리
- `packages/opencode/src/tool/registry.ts` — 도구 레지스트리

## 개요

도구 시스템은 세 계층으로 구성된다:

1. **Tool.define()**: 개별 도구를 정의하는 팩토리 함수(Factory function)
2. **ToolRegistry**: 도구를 등록하고 모델/에이전트별로 필터링하여 제공
3. **MCP 도구**: 외부 MCP 서버에서 가져온 도구를 내장 도구와 동일하게 취급

## Tool.define()

```typescript
// packages/opencode/src/tool/tool.ts
export namespace Tool {
  export interface Info {
    id: string
    init(): Promise<{
      description: string
      parameters: z.ZodObject<any>
      execute(args: any, ctx: Context): Promise<string>
    }>
  }

  export interface Context {
    sessionID: string
    messageID: string
    agent: Agent.Info
    abort: AbortSignal
    callID: string
    messages: MessageV2.WithParts[]
    metadata: (meta: Record<string, unknown>) => void
    ask: PermissionNext.ask  // 권한 요청 함수
  }

  export function define<P extends z.ZodObject<any>, M>(options: {
    id: string
    init(): Promise<{
      description: string
      parameters: P
      execute(args: z.infer<P>, ctx: Context): Promise<string>
    }>
  }): Tool.Info
}
```

### define()의 내부 동작

1. `init()`을 호출하여 도구 설명, 파라미터 스키마, 실행 함수를 가져옴
2. 실행 시 Zod 스키마로 입력 인자 검증
3. `ZodError` 발생 시 커스텀 포매터로 에러 메시지 생성
4. 출력이 너무 길면 `Truncate.output()`으로 자동 잘라냄
5. 메타데이터의 `truncated` 플래그가 설정된 경우 잘라내기 생략

## 내장 도구 목록

| 도구 | ID | 설명 |
|------|-----|------|
| Bash | `bash` | 셸 명령 실행 |
| Read | `read` | 파일 읽기 |
| Write | `write` | 파일 쓰기 |
| Edit | `edit` | 파일 편집 (문자열 교체) |
| Glob | `glob` | 파일 패턴 검색 |
| Grep | `grep` | 파일 내용 검색 |
| WebFetch | `webfetch` | URL 내용 가져오기 |
| WebSearch | `websearch` | 웹 검색 |
| CodeSearch | `codesearch` | 코드 검색 |
| Task | `task` | 하위 에이전트 실행 |
| TodoWrite | `todowrite` | TODO 항목 관리 |
| Skill | `skill` | 스킬 실행 |
| ApplyPatch | `applypatch` | 통합 diff 패치 적용 |
| LSP | `lsp` | Language Server Protocol 연동 |
| Batch | `batch` | 여러 도구 동시 실행 |
| PlanEnter | `planenter` | 계획 모드 진입 |
| PlanExit | `planexit` | 계획 모드 종료 |
| Question | `question` | 사용자에게 질문 |
| InvalidTool | `invalid` | 잘못된 도구 호출 시 에러 반환 |

## ToolRegistry

도구 레지스트리는 도구를 등록하고, 모델·에이전트·설정에 따라 적절한 도구 목록을 반환한다.

```typescript
// packages/opencode/src/tool/registry.ts
export namespace ToolRegistry {
  // 도구 등록
  export function register(tool: Tool.Info): void

  // 모델과 에이전트에 맞는 도구 목록 반환 (초기화 완료)
  export function tools(
    model: Provider.Model,
    agent?: Agent.Info
  ): Promise<Record<string, Tool>>

  // 등록된 도구 ID 목록
  export function ids(): string[]
}
```

### 도구 선택 로직

`ToolRegistry.tools()`는 조건에 따라 도구를 필터링한다:

| 도구 | 조건 |
|------|------|
| WebSearch, CodeSearch | OpenCode 프로바이더 또는 `OPENCODE_ENABLE_EXA` 플래그 |
| ApplyPatch | GPT-5 이상 모델만 (나머지는 Edit/Write 사용) |
| LSP | `OPENCODE_EXPERIMENTAL_LSP_TOOL` 플래그 |
| Batch | `config.experimental.batch_tool = true` |
| PlanEnter, PlanExit | 플래그 + CLI 클라이언트에서만 |
| Question | app/cli/desktop 클라이언트에서만 |

### 커스텀 도구 로딩

`Instance.state()`를 통해 커스텀 도구를 지연 로딩한다:

```typescript
const state = Instance.state(async () => {
  const custom: Tool.Info[] = []

  // Config.directories() 내의 tool/ 또는 tools/ 디렉터리 스캔
  for (const dir of Config.directories()) {
    for (const pattern of ["tool", "tools"]) {
      const files = glob.sync(`${dir}/${pattern}/*.{js,ts}`)
      for (const file of files) {
        const tool = await import(file)
        custom.push(tool.default)
      }
    }
  }

  return { custom }
})
```

### 플러그인 도구 변환

플러그인에서 정의한 도구는 `PluginToolDefinition` → `Tool.Info`로 변환된다:

```typescript
// 플러그인 도구를 내부 형식으로 변환
function fromPlugin(def: PluginToolDefinition): Tool.Info {
  return {
    id: def.name,
    async init() {
      return {
        description: def.description,
        parameters: def.parameters,
        async execute(args, ctx) {
          return def.execute(args, {
            // 컨텍스트 매핑
            sessionID: ctx.sessionID,
            agent: ctx.agent,
            // ...
          })
        },
      }
    },
  }
}
```

## 도구 실행 흐름

```
LLM이 도구 호출 요청
    │
    ▼
ToolRegistry에서 도구 검색
    │
    ├── 내장 도구 → Tool.Info.execute()
    ├── MCP 도구 → MCP 클라이언트로 전달
    └── 커스텀 도구 → 로드된 모듈 실행
    │
    ▼
PermissionNext.ask() — 권한 확인
    │
    ├── allow → 실행
    ├── deny → DeniedError
    └── ask → UI에서 사용자 승인 대기
    │
    ▼
실행 결과 → Truncate.output() → LLM에 반환
```

## 도구 상태 머신

`ToolPart`는 다음 상태를 거친다:

```
pending → running → completed
                  → error
```

각 상태 전환 시 `MessageV2.Event.PartUpdated`가 발행되어 UI가 실시간으로 도구 실행 상태를 표시할 수 있다.

## 참고 문서

- [05. 세션 생명주기](./05-session-lifecycle.md) — 도구가 세션 처리 루프에서 실행되는 방식
- [09. 권한 시스템](./09-permission-system.md) — 도구 권한 확인
- [13. MCP 통합](./13-mcp-integration.md) — MCP 도구
- [19. 도구 추가하기](./19-adding-a-tool.md) — 새 도구 추가 가이드
