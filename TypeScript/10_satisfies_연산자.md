### 10. satisfies 연산자

<br>

#### 1. 등장 배경 - 검증과 추론은 항상 같이 가지 않는다

---

- 객체 리터럴을 작성할 때 개발자는 보통 두 가지를 동시에 원함
  1. 이 객체가 특정 타입의 조건을 **만족하는지 검증**하고 싶음 (오타, 누락된 키, 잘못된 값 타입을 컴파일 타임에 잡고 싶음)
  2. 검증이 끝난 뒤에는 각 프로퍼티의 **가장 구체적인 타입**을 그대로 쓰고 싶음 (자동완성, 타입 좁히기 등)
- TS 4.9 이전에는 이 둘을 동시에 만족시키는 방법이 마땅치 않았음 — 타입 애너테이션을 쓰면 1번은 되지만 2번을 잃고, 아무것도 안 쓰면 2번은 되지만 1번을 잃음
- `satisfies`는 이 둘을 **동시에** 만족시키기 위해 TS 4.9에서 도입된 연산자

<br>

#### 2. 타입 애너테이션의 한계

---

- 흔히 쓰는 방식은 변수에 타입을 직접 명시하는 것

```typescript
type Colors = "red" | "green" | "blue";
type RGB = [red: number, green: number, blue: number];

const palette: Record<Colors, string | RGB> = {
  red: [255, 0, 0],
  green: "#00ff00",
  bleu: [0, 0, 255], // ✅ 오타를 컴파일 타임에 잡아줌 ("bleu"는 Colors에 없는 키)
};

const greenNormalized = palette.green.toUpperCase();
// ❌ 에러: green의 타입이 string | RGB로 넓혀져서, RGB에는 toUpperCase가 없다는 이유로 거부됨
```

- `palette: Record<Colors, string | RGB>`라고 애너테이션을 붙이는 순간, TS는 각 프로퍼티의 실제 값이 무엇이었는지는 잊어버리고 선언된 타입(`string | RGB`)으로만 취급함
- 즉 `green`이 실제로는 `string`이었다는 구체적인 정보가 애너테이션 단계에서 날아가 버림 — 07번 글에서 다룬 `as const`가 "값을 그대로 유지하되 검증은 포기"하는 것과 정반대로, 애너테이션은 "검증은 하되 구체적인 값 정보를 포기"하는 셈

<br>

#### 3. satisfies로 검증과 추론을 둘 다 챙기기

---

- 변수 선언부가 아니라 값 뒤에 `satisfies Type`을 붙이면, **타입 검증은 하되 변수의 타입은 원래 표현식에서 추론된 대로 유지**됨

```typescript
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  bleu: [0, 0, 255], // ✅ 여전히 오타 감지됨
} satisfies Record<Colors, string | RGB>;

const greenNormalized = palette.green.toUpperCase();
// ✅ 정상 동작 - green이 string으로 정확히 추론됨
const redComponent = palette.red[0];
// ✅ red가 RGB(튜플)로 추론되어 인덱스 접근도 안전함
```

- 동작 순서로 이해하면 편함
  1. `{ red: [...], green: "...", bleu: [...] }` 라는 객체 리터럴 자체의 타입을 먼저 추론
  2. 그 추론된 타입이 `Record<Colors, string | RGB>`를 만족하는지(assignable한지) **검사만** 하고, 통과하면 검사에 쓴 타입은 버림
  3. 최종적으로 변수의 타입은 1번에서 추론된 원래 타입이 됨
- 그래서 `bleu`처럼 `Colors`에 없는 키를 넣거나, `blue: [0, 0]`처럼 튜플 길이가 안 맞으면 여전히 컴파일 에러가 남 — 검증 기능은 애너테이션과 동일하게 살아있음

<br>

#### 4. as, 타입 애너테이션과 satisfies의 차이

---

- 셋 다 "이 값을 특정 타입으로 다루고 싶다"는 상황에서 쓰이지만 성격이 완전히 다름

| 방식 | 컴파일 타임 검증 | 결과 타입 |
| --- | --- | --- |
| `const x: T = value` (애너테이션) | O | `T` (선언한 타입으로 넓혀짐) |
| `value as T` (단언) | 사실상 없음 | `T` (컴파일러에게 "믿어달라"고 강제로 우기는 것) |
| `value satisfies T` | O | `value`의 원래 추론 타입 (가장 구체적인 타입 유지) |

- `as`는 컴파일러의 타입 추론을 **강제로 무시**시키는 단언이라, 03번 글의 제네릭 예제에서 언급했듯 실제 값과 단언한 타입이 다르면 런타임 에러로 이어질 수 있어 지양되는 편임
- `satisfies`는 반대로 컴파일러가 **직접 검증한 뒤** 원래 추론 결과를 돌려주는 것이라 더 안전함

> 찰떡 비유: 신분증 검사
>
> `as`가 "내 나이는 20살이야"라고 우겨서 컴파일러가 별다른 확인 없이 믿어버리는 것이라면, `satisfies`는 실제 신분증(원래 값)을 보여주고 "성인 조건(타입)을 만족하는지" 검사받은 뒤, 검사가 끝나면 생년월일이 지워진 "성인" 카드가 아니라 원래 신분증(구체적인 타입 정보)을 그대로 돌려받는 것과 같음

<br>

#### 5. as const와 함께 쓰기 - `as const satisfies T`

---

- `as const`는 값을 리터럴 타입으로 좁히고 `readonly`로 만들지만 검증은 하지 않고, `satisfies`는 검증은 하지만 그 자체로 `readonly`를 만들어주지는 않음 — 둘을 합치면 **검증 + 리터럴 타입 유지 + 불변성**을 동시에 얻을 수 있음
- 순서는 반드시 `as const`가 먼저, `satisfies`가 나중

```typescript
type Metadata = string | number | { [key: string]: Metadata };
type Profile = Record<string, Metadata>;

const profile = {
  name: "topher",
  age: 2,
  status: "active",
} as const satisfies Profile;

profile.status; // "active" (string이 아니라 리터럴 "active"로 추론됨)
profile.age; // 2 (number가 아니라 리터럴 2)

// profile.age = 3;
// ❌ 에러: as const로 readonly가 되어 재할당 불가
```

- `as const`만 썼다면 `Profile`을 만족하는지 검증이 안 되고, `satisfies`만 썼다면 `age`가 `2`가 아니라 `number`로 넓혀짐 — 둘을 겹쳐야 세 가지 이점을 모두 챙길 수 있음

<br>

#### 6. 언제 쓰면 좋은가

---

- 라우트 맵, 테마 색상표, i18n 메시지 객체처럼 **키/값 구조는 검증하고 싶지만, 이후 코드에서는 각 값의 구체적인 타입(리터럴 문자열, 튜플 등)을 그대로 쓰고 싶은 설정성 객체**에 잘 맞음
- 함수 인자나 반환값처럼 "이 자리는 무조건 이 타입으로 취급하고 싶다"는 경우에는 오히려 일반 애너테이션이 더 적합할 수 있음 — `satisfies`는 "정의하는 값 자체의 구체적인 타입을 살리고 싶을 때" 쓰는 도구지, 애너테이션을 완전히 대체하는 개념은 아님
- 반대로 값이 실제로 해당 타입과 다를 수 있다는 걸 컴파일러가 검증할 수 없는 상황(외부 API 응답을 강제로 캐스팅하는 등)에서는 `satisfies`로 해결이 안 되고 `as`나 런타임 검증(파서/스키마 검증 라이브러리)이 필요함
