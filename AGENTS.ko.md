- SDK를 재생성하려면 `./packages/sdk/js/script/build.ts`를 실행하세요.
- 적용 가능한 경우 항상 병렬 도구를 사용하세요.
- 이 저장소의 기본 브랜치(Branch)는 `dev`입니다.
- 로컬 `main` 참조가 없을 수 있으므로 diff에는 `dev` 또는 `origin/dev`를 사용하세요.
- 자동화를 선호합니다: 정보 부족이나 안전/비가역성 문제로 차단되지 않는 한 확인 없이 요청된 작업을 수행하세요.

## 스타일 가이드

[English version](./AGENTS.md)

### 일반 원칙

- 합성 가능하거나 재사용 가능하지 않은 한 하나의 함수에 로직을 유지
- 가능하면 `try`/`catch` 사용을 피함
- `any` 타입 사용을 피함
- 가능하면 단일 단어 변수명 선호
- `Bun.file()` 등 Bun API를 적극 사용
- 가능하면 타입 추론에 의존하고 명시적 타입 어노테이션이나 인터페이스는 export나 명확성을 위해 필요한 경우에만 사용
- for 루프보다 함수형 배열 메서드(flatMap, filter, map) 선호; filter에서 타입 가드를 사용하여 다운스트림 타입 추론 유지

### 명명 규칙

변수와 함수 이름은 단일 단어를 선호합니다. 필요한 경우에만 복수 단어를 사용하세요.

```ts
// 좋은 예
const foo = 1
function journal(dir: string) {}

// 나쁜 예
const fooBar = 1
function prepareJournal(dir: string) {}
```

값이 한 번만 사용되는 경우 인라인으로 변수 수를 줄이세요.

```ts
// 좋은 예
const journal = await Bun.file(path.join(dir, "journal.json")).json()

// 나쁜 예
const journalPath = path.join(dir, "journal.json")
const journal = await Bun.file(journalPath).json()
```

### 구조 분해(Destructuring)

불필요한 구조 분해를 피하세요. 컨텍스트를 유지하기 위해 점 표기법을 사용하세요.

```ts
// 좋은 예
obj.a
obj.b

// 나쁜 예
const { a, b } = obj
```

### 변수

`let`보다 `const`를 선호합니다. 재할당 대신 삼항 연산자나 조기 반환을 사용하세요.

```ts
// 좋은 예
const foo = condition ? 1 : 2

// 나쁜 예
let foo
if (condition) foo = 1
else foo = 2
```

### 제어 흐름

`else` 문을 피하세요. 조기 반환(early return)을 선호합니다.

```ts
// 좋은 예
function foo() {
  if (condition) return 1
  return 2
}

// 나쁜 예
function foo() {
  if (condition) return 1
  else return 2
}
```

### 스키마 정의 (Drizzle)

필드명은 snake_case를 사용하여 컬럼명을 문자열로 재정의할 필요가 없도록 합니다.

```ts
// 좋은 예
const table = sqliteTable("session", {
  id: text().primaryKey(),
  project_id: text().notNull(),
  created_at: integer().notNull(),
})

// 나쁜 예
const table = sqliteTable("session", {
  id: text("id").primaryKey(),
  projectID: text("project_id").notNull(),
  createdAt: integer("created_at").notNull(),
})
```

## 테스트

- 모의 객체(Mock)를 최대한 피하세요
- 실제 구현을 테스트하고, 로직을 테스트에 중복시키지 마세요
