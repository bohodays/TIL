### 08. 조건부 타입과 infer

---

#### 1. 조건부 타입(Conditional Types)이란

---

- `T extends U ? X : Y` 형태로, 타입 레벨에서 삼항 연산자처럼 분기를 만드는 문법
  - 런타임 코드가 아니라 **컴파일 타임에 타입만 결정**된다는 점이 일반 삼항 연산자와 다름
- `T`가 `U`에 할당 가능하면(`assignable`) `X`, 아니면 `Y` 타입이 됨

```typescript
type IsString<T> = T extends string ? true : false;

type A = IsString<"hello">; // true
type B = IsString<123>; // false
```

- 제네릭과 결합하면 "받은 타입에 따라 다른 타입을 돌려주는 함수"를 타입 시스템 위에서 만들 수 있음
- 실무에서 직접 `IsString` 같은 걸 쓸 일은 적지만, 라이브러리 타입 정의(`.d.ts`)나 유틸리티 타입 내부는 대부분 이 패턴으로 되어 있음

<br>

#### 2. infer 키워드로 타입 추출하기

---

- `infer`는 조건부 타입의 `extends` 절 안에서만 쓸 수 있는 키워드
- "이 자리에 오는 타입이 뭔지 모르겠지만, 일단 변수로 잡아서(`infer`) 참/거짓 분기 안에서 쓰겠다"는 의미
- 함수의 반환 타입을 뽑아내는 예시가 가장 직관적

```typescript
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

function getUser() {
  return { name: "ellie", age: 20 };
}

type User = MyReturnType<typeof getUser>;
// { name: string; age: number }
```

- `T extends (...args: any[]) => infer R`
  - "T가 함수 형태라면, 그 반환 타입을 R이라는 이름으로 잡아둬라"는 뜻
  - 실제로 함수를 호출하는 게 아니라, **함수의 타입 모양(shape)만 보고 반환 타입 부분만 떼어오는 것**
- TS 내장 유틸리티 타입인 `ReturnType<T>`가 정확히 이 구조로 구현되어 있음
  - `lib.es5.d.ts`를 열어보면 `type ReturnType<T extends (...args: any) => any> = T extends (...args: any) => infer R ? R : any;` 형태를 그대로 확인할 수 있음

<br>

#### 3. 분산 조건부 타입 (Distributive Conditional Types)

---

- 조건부 타입의 `T` 자리가 **아무 수식도 없는 제네릭 타입 파라미터 그 자체**(naked type parameter)이고, 여기에 **유니온 타입**을 넣으면 조건부 타입이 유니온의 각 멤버에 대해 따로 실행된 뒤 다시 합쳐짐
  - 배열을 순회하며 각 원소에 필터를 적용하고 결과를 다시 배열로 모으는 것과 비슷한 동작

```typescript
type ToArray<T> = T extends any ? T[] : never;

type StrOrNumArray = ToArray<string | number>;
// string[] | number[]   (❌ (string | number)[] 아님)
```

- 동작 순서
  1. `string | number`가 `ToArray`에 들어감
  2. 유니온이 `string`과 `number`로 쪼개져서 각각 `ToArray<string>`, `ToArray<number>`가 따로 평가됨
  3. 결과인 `string[]`과 `number[]`가 다시 유니온으로 합쳐짐
- `Exclude<T, U>`, `Extract<T, U>` 같은 내장 유틸리티 타입도 이 분산 성질을 이용해서 구현되어 있음
  ```typescript
  type MyExclude<T, U> = T extends U ? never : T;

  type A = MyExclude<"a" | "b" | "c", "a">; // "b" | "c"
  ```
  - `"a" | "b" | "c"`가 하나씩 분산되면서 `U`("a")와 일치하는 멤버만 `never`가 되고, `never`는 유니온에서 자동으로 사라지기 때문에 결과적으로 제외 효과가 남

<br>

#### 4. 분산을 막고 싶을 때 - [T] extends [U]

---

- 항상 분산되는 게 편한 건 아님. 유니온을 쪼개지 않고 **하나의 통짜 타입으로** 비교하고 싶을 때가 있음
- `T`, `U`를 튜플로 한 번 감싸면(`[T] extends [U]`) naked type parameter 조건이 깨지기 때문에 분산이 일어나지 않음

```typescript
type IsUnion<T> = [T] extends [any] ? false : never; // 예시 목적의 단순화

type NaiveCheck<T> = T extends string | number ? "yes" : "no";
type A = NaiveCheck<string>; // "yes" (분산되어 개별 판단)

type WrappedCheck<T> = [T] extends [string | number] ? "yes" : "no";
type B = WrappedCheck<string>; // "yes" (분산 안 됨, 전체를 한 번에 비교)
```

- 두 결과가 위 예시처럼 같아 보여도, 유니온을 인자로 넘겼을 때는 분산 여부에 따라 결과가 완전히 달라짐
  - 분산형: 유니온의 각 멤버를 하나씩 검사 → 부분적으로 걸러내는 데 적합
  - 비분산형(`[T] extends [U]`): 유니온 전체를 하나의 타입으로 취급 → "이 타입이 통째로 무엇인가"를 판단할 때 적합

<br>

#### 5. 실전 감각 - Awaited\<T\>는 왜 재귀적일까

---

- `Promise<Promise<string>>`처럼 프로미스가 중첩돼도 `Awaited<T>`는 최종적으로 `string`을 뽑아냄
- 이건 조건부 타입이 **재귀적으로 자기 자신을 호출**할 수 있기 때문에 가능

```typescript
type MyAwaited<T> = T extends Promise<infer U> ? MyAwaited<U> : T;

type A = MyAwaited<Promise<Promise<string>>>; // string
type B = MyAwaited<number>; // number (Promise가 아니면 그대로 반환)
```

- `T`가 `Promise<infer U>`와 매칭되는 동안은 `U`를 다시 `MyAwaited`에 넣어 계속 벗겨내고, 더 이상 `Promise`가 아니게 되면 그 시점의 타입을 그대로 반환
- 함수 재귀와 마찬가지로 **기저 조건(base case)**이 없으면 무한 재귀로 컴파일 에러가 나므로, `T extends Promise<infer U> ? ... : T` 처럼 반드시 탈출 분기를 둬야 함
