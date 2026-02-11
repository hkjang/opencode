# 19. 도구 추가하기

이 문서는 OpenCode에 새 도구(Tool)를 추가하는 실전 가이드이다.

**핵심 파일:**
- `packages/opencode/src/tool/tool.ts` — 도구 인터페이스
- `packages/opencode/src/tool/registry.ts` — 도구 레지스트리

## 개요

새 도구를 추가하는 방법은 세 가지이다:

1. **내장 도구**: 코어 패키지에 직접 추가 (PR 필요)
2. **커스텀 도구**: 설정 디렉터리의 `tool/` 폴더에 파일 추가
3. **플러그인 도구**: `@opencode-ai/plugin` SDK로 플러그인에 포함

## 방법 1: 내장 도구 추가

### 단계 1: 도구 파일 생성

`packages/opencode/src/tool/` 디렉터리에 새 파일을 만든다.

```typescript
// packages/opencode/src/tool/mytool.ts
import z from "zod"
import { Tool } from "./tool"
import { Instance } from "../project/instance"

export const MyTool = Tool.define("mytool", {
  // 도구 설명 (LLM이 도구 선택 시 참고)
  description: "이 도구의 용도를 설명합니다. LLM이 언제 이 도구를 사용해야 하는지 명확히 기술하세요.",

  // Zod 스키마로 파라미터 정의
  parameters: z.object({
    input: z.string().describe("입력 값에 대한 설명"),
    option: z.boolean().optional().describe("선택적 옵션"),
  }),

  // 실행 함수
  async execute(params, ctx) {
    // 1. 권한 요청
    await ctx.ask({
      permission: "mytool",          // 권한 이름
      patterns: [params.input],      // 매칭할 패턴
      always: ["*"],                 // "always" 승인 시 적용할 패턴
      metadata: {                    // UI에 표시할 메타데이터
        input: params.input,
      },
    })

    // 2. 취소 확인
    if (ctx.abort.aborted) {
      throw new Error("중단됨")
    }

    // 3. 비즈니스 로직
    const result = await doSomething(params.input)

    // 4. 진행 상태 업데이트 (선택)
    ctx.metadata({
      title: "처리 중...",
      metadata: { progress: 50 },
    })

    // 5. 결과 반환
    return {
      title: "MyTool 결과",            // UI에 표시할 제목
      metadata: { truncated: false },  // truncated: true면 잘라내기 생략
      output: result,                  // LLM에 전달할 텍스트 결과
    }
  },
})
```

### 단계 2: 도구 설명 파일 생성 (선택)

긴 설명은 별도 텍스트 파일로 분리한다.

```
// packages/opencode/src/tool/mytool.txt
이 도구는 XYZ를 수행합니다.

사용 시점:
- 조건 A일 때
- 조건 B일 때

파라미터:
- input: 처리할 입력 값
- option: true이면 추가 처리 수행
```

```typescript
// mytool.ts에서 import
import DESCRIPTION from "./mytool.txt"

export const MyTool = Tool.define("mytool", {
  description: DESCRIPTION,
  // ...
})
```

### 단계 3: 레지스트리에 등록

```typescript
// packages/opencode/src/tool/registry.ts
import { MyTool } from "./mytool"

async function all(): Promise<Tool.Info[]> {
  return [
    // 기존 도구들...
    BashTool,
    ReadTool,
    GlobTool,
    // ...

    // 새 도구 추가
    MyTool,

    // 조건부 추가도 가능
    // ...(someCondition ? [MyTool] : []),

    ...custom,
  ]
}
```

### 단계 4: 권한 기본값 설정 (선택)

에이전트별로 도구의 기본 권한을 설정할 수 있다.

```typescript
// packages/opencode/src/agent/agent.ts
// 에이전트의 기본 권한에 추가
const buildDefaults = [
  // ...기존 규칙
  { permission: "mytool", pattern: "*", action: "ask" },
]

// plan 에이전트에서 비활성화
const planDefaults = [
  // ...기존 규칙
  { permission: "mytool", pattern: "*", action: "deny" },
]
```

## 방법 2: 커스텀 도구 (설정 디렉터리)

설정 디렉터리에 파일을 추가하면 코어 코드를 수정하지 않고도 도구를 추가할 수 있다.

### 파일 위치

```
~/.config/opencode/
├── tool/              # 또는 tools/
│   └── my-tool.ts     # 커스텀 도구
```

또는 프로젝트의 `.opencode/` 디렉터리:

```
.opencode/
├── tool/
│   └── my-tool.ts
```

### 커스텀 도구 파일

```typescript
// ~/.config/opencode/tool/my-tool.ts
import z from "zod"

export default {
  name: "my-custom-tool",
  description: "커스텀 도구 설명",
  parameters: z.object({
    query: z.string(),
  }),
  async execute(args: { query: string }, ctx: any) {
    return `결과: ${args.query}`
  },
}
```

커스텀 도구는 자동으로 `ToolRegistry`에 로드된다. 파일명이 도구 이름의 네임스페이스가 된다.

## 방법 3: 플러그인 도구

`@opencode-ai/plugin` SDK를 사용하여 플러그인 내에서 도구를 정의한다.

```typescript
// my-plugin/index.ts
import { definePlugin } from "@opencode-ai/plugin"
import { defineTool } from "@opencode-ai/plugin/tool"
import z from "zod"

const myTool = defineTool({
  name: "plugin-tool",
  description: "플러그인 도구 설명",
  parameters: z.object({
    input: z.string(),
  }),
  async execute(args, ctx) {
    // ctx.client로 OpenCode SDK 사용 가능
    return `결과: ${args.input}`
  },
})

export default definePlugin({
  name: "my-plugin",
  tool: {
    "plugin-tool": myTool,
  },
})
```

## Tool.Context 상세

도구 실행 시 제공되는 컨텍스트:

```typescript
interface Context {
  sessionID: string              // 현재 세션 ID
  messageID: string              // 현재 메시지 ID
  agent: string                  // 에이전트 이름
  abort: AbortSignal             // 취소 시그널
  callID?: string                // 도구 호출 ID
  messages: MessageV2.WithParts[] // 현재 대화 메시지 목록
  extra?: Record<string, any>    // 추가 데이터

  // 메타데이터 업데이트 (UI에 진행 상태 표시)
  metadata(input: {
    title?: string
    metadata?: Record<string, unknown>
  }): void

  // 권한 요청
  ask(input: {
    permission: string           // 권한 이름
    patterns: string[]           // 매칭할 패턴
    always?: string[]            // "always" 승인 시 패턴
    metadata: Record<string, unknown>
  }): Promise<void>
}
```

## 반환값 구조

```typescript
{
  title: string                        // UI에 표시할 짧은 제목
  metadata: {
    truncated?: boolean                // true면 출력 잘라내기 생략
    [key: string]: unknown             // 커스텀 메타데이터
  }
  output: string                       // LLM에 전달할 텍스트 결과
  attachments?: MessageV2.FilePart[]   // 이미지, PDF 등 첨부파일 (선택)
}
```

## 출력 잘라내기

기본적으로 도구 출력이 너무 길면 `Truncate.output()`이 자동으로 잘라낸다. 이를 제어하려면:

- `metadata.truncated = false` — 잘라내기를 허용 (기본값)
- `metadata.truncated = true` — 잘라내기를 직접 처리했으므로 생략

## 실전 패턴: Glob 도구 분석

실제 코드베이스의 간결한 도구 구현 예시:

```typescript
// packages/opencode/src/tool/glob.ts (핵심 구조)
export const GlobTool = Tool.define("glob", {
  description: DESCRIPTION,

  parameters: z.object({
    pattern: z.string().describe("매칭할 글로브 패턴"),
    path: z.string().optional().describe("검색할 디렉터리"),
  }),

  async execute(params, ctx) {
    // 권한 요청
    await ctx.ask({
      permission: "glob",
      patterns: [params.pattern],
      always: ["*"],
      metadata: { pattern: params.pattern, path: params.path },
    })

    // 경로 해석
    let search = params.path ?? Instance.directory
    search = path.isAbsolute(search)
      ? search
      : path.resolve(Instance.directory, search)

    // 실행
    const files = []
    for await (const file of Ripgrep.files({
      cwd: search,
      glob: [params.pattern],
      signal: ctx.abort,
    })) {
      if (files.length >= 100) break
      files.push(path.resolve(search, file))
    }

    // 결과 반환
    return {
      title: path.relative(Instance.worktree, search),
      metadata: { count: files.length, truncated: false },
      output: files.join("\n"),
    }
  },
})
```

## 체크리스트

- [ ] `Tool.define()`으로 도구 정의
- [ ] Zod 스키마로 파라미터 정의 (`.describe()`로 설명 추가)
- [ ] `ctx.ask()`로 권한 요청 구현
- [ ] `ctx.abort` 시그널 확인
- [ ] `ToolRegistry`의 `all()`에 등록 (내장 도구의 경우)
- [ ] 반환값에 `title`, `metadata`, `output` 포함
- [ ] 도구 설명이 LLM이 이해하기 충분히 명확한지 확인
- [ ] 에이전트별 권한 기본값 설정 (필요 시)
- [ ] 조건부 도구인 경우 `registry.ts`에 조건 추가

## 참고 문서

- [07. 도구 시스템](./07-tool-system.md) — 도구 시스템 아키텍처
- [09. 권한 시스템](./09-permission-system.md) — `ctx.ask()` 동작 방식
- [12. 플러그인 시스템](./12-plugin-system.md) — 플러그인 도구
- [CONTRIBUTING.ko.md](../../../CONTRIBUTING.ko.md) — PR 규칙
