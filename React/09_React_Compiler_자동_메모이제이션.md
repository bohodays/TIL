### 09. React Compiler (자동 메모이제이션)

<br>

#### 1. 왜 필요한가 — 수동 메모이제이션의 한계

---

- 이전에 정리했던 [`React.memo`](https://github.com/bohodays/TIL/blob/master/React/04_React_memo%EB%A1%9C_%EC%B5%9C%EC%A0%81%ED%99%94.md), [`useMemo`](https://github.com/bohodays/TIL/blob/master/React/05_%EB%B6%88%ED%95%84%EC%9A%94%ED%95%9C_%EA%B3%84%EC%82%B0%EC%9D%84_%EC%A4%84%EC%9D%B4%EB%8A%94_useMemo.md)는 전부 **개발자가 직접** "여기서 재렌더링이 낭비다"를 판단해서 붙이는 수동 최적화다.
- 문제
  - 어디에 붙여야 할지 사람이 매번 판단해야 하고, 의존성 배열을 하나만 빠뜨려도 버그가 된다.
  - `useCallback`으로 함수를 감싸도, 그 함수를 받는 자식이 `React.memo`로 감싸져 있지 않으면 아무 효과가 없다 — 최적화가 "체인"으로 연결되어 있어야만 동작한다.
  - React는 기본적으로 **부모가 리렌더링되면 자식도 전부 리렌더링**한다(cascading re-render). props가 안 바뀌어도 그렇다.
- React Compiler는 이 판단을 사람이 아니라 **빌드타임 정적 분석**으로 옮긴다. 컴포넌트 코드를 순수 JS로 작성하면, 컴파일러가 "이 값은 렌더마다 다시 계산할 필요가 없다"를 스스로 찾아내서 메모이제이션 코드를 자동으로 끼워 넣는다.

<br>

#### 2. 어떻게 동작하는가 — Reactive Scope 분석

---

- React Compiler는 Babel 플러그인(Next.js 15.3.1+는 SWC 기반)으로 빌드 시점에 컴포넌트/훅 함수를 분석한다.
- 컴포넌트 안의 각 표현식(변수, JSX 조각 등)을 **reactive scope** 단위로 쪼개서, 어떤 값이 어떤 입력(props/state/context)에 의존하는지 데이터 흐름을 추적한다.
  - 훅 단위가 아니라 **표현식 단위**로 캐시 경계를 나눈다는 점이 핵심이다. 사람이 손으로 `useMemo`를 쓰면 보통 함수 하나 통째로 감싸지만, 컴파일러는 그보다 훨씬 세밀하게(fine-grained) 쪼개서 필요한 부분만 재계산한다.
  - 내부적으로는 각 컴포넌트마다 숨겨진 캐시 슬롯(`useMemoCache`)을 만들어, 이전 렌더의 의존값과 비교해 바뀌지 않았으면 이전 결과를 그대로 재사용하는 코드를 생성한다.
- 즉, `useMemo`/`useCallback`/`React.memo`를 쓴 것과 **결과적으로 같은 일**을 하지만, 사람이 의존성 배열을 나열할 필요가 없고 누락 버그도 생기지 않는다.

```jsx
// 사람이 직접 작성하는 코드 (컴파일러 없이도, 있어도 동일하게 작성)
function FriendList({ friends }) {
  const onlineCount = useFriendOnlineCount();
  return (
    <div>
      <span>{onlineCount} online</span>
      {friends.map((friend) => (
        <FriendListCard key={friend.id} friend={friend} />
      ))}
      {/* onlineCount만 바뀌어도 원래는 MessageButton까지 리렌더링됨 */}
      <MessageButton />
    </div>
  );
}
```

- 위 코드는 `useMemo`/`memo`가 하나도 없지만, 컴파일러를 거치면 `onlineCount`가 바뀌어도 `friends`에 의존하지 않는 `MessageButton` 쪽 트리는 재사용된다. **코드는 그대로 두고, 컴파일 결과물만 최적화되는 구조**다.

<br>

#### 3. 수동 메모이제이션이 놓치던 버그도 함께 잡아준다

---

```jsx
// ❌ 수동 최적화: useCallback을 썼지만 사실상 무의미
const ExpensiveComponent = memo(function ExpensiveComponent({ data, onClick }) {
  const processedData = useMemo(() => expensiveProcessing(data), [data]);

  const handleClick = useCallback((item) => onClick(item.id), [onClick]);

  return (
    <div>
      {processedData.map((item) => (
        // 화살표 함수가 렌더마다 새로 생성되어, 위 useCallback은 무력화된다
        <Item key={item.id} onClick={() => handleClick(item)} />
      ))}
    </div>
  );
});
```

```jsx
// ✅ 컴파일러 활성화 시: 순수 로직만 남기면 됨
function ExpensiveComponent({ data, onClick }) {
  const processedData = expensivelyProcessAReallyLargeArrayOfObjects(data);

  const handleClick = (item) => onClick(item.id);

  return (
    <div>
      {processedData.map((item) => (
        <Item key={item.id} onClick={() => handleClick(item)} />
      ))}
    </div>
  );
}
```

- `useCallback`으로 감쌌든 안 감쌌든, `() => handleClick(item)`처럼 렌더마다 새로 만들어지는 인라인 함수는 컴파일러가 알아서 안전하게 처리한다. 사람이 짠 수동 메모이제이션은 오히려 "형식만 맞추고 실제로는 동작 안 하는" 죽은 코드가 되기 쉬운데, 컴파일러는 이런 착시가 없다.

<br>

#### 4. 전제조건 — Rules of React

---

- 컴파일러는 마법이 아니라 "React 컴포넌트가 원래 지켜야 할 규칙"을 코드가 실제로 지키고 있다는 전제 위에서 안전하게 최적화한다. 그 규칙이 **Rules of React**다.
  - **순수성(Idempotency)**: 같은 props/state/context가 들어오면 항상 같은 JSX를 반환해야 한다.
  - **렌더 중 부수효과 금지**: fetch, 구독, DOM 조작 같은 부수효과는 렌더 함수 밖(`useEffect` 등)에서 실행해야 한다.
  - **불변성**: props와 state는 해당 렌더 시점의 스냅샷이다. 직접 `mutate`하면 안 된다.
  - **훅은 조건문/반복문 없이 항상 같은 순서로 호출**해야 한다.
- 이 규칙들을 지키지 않는 코드(예: 렌더 중에 props를 직접 변경)는 컴파일러가 있든 없든 원래도 버그였지만, 지금까지는 우연히 동작하는 경우가 많았다. 컴파일러를 켜면 "우연히 동작하던" 코드가 실제로 잘못 최적화되어 드러날 수 있다.

<br>

#### 5. Bailout — 최적화를 포기하는 경우

---

- 컴파일러는 어떤 스코프가 Rules of React를 위반한다고 판단되면, **틀린 코드를 만들 바에는 그냥 최적화를 포기**한다. 이걸 bailout이라고 한다.
- 대표적인 bailout 상황
  - 클래스 인스턴스에 의존하는 값 — 클래스 인스턴스 생성 자체를 메모할 수 없어서, 그 인스턴스에 의존하는 코드 전체가 최적화 대상에서 빠진다.
  - early return 앞에 훅을 안전하게 배치할 수 없는 경우
  - 남겨둔 `useMemo`/`useCallback`의 의존성이 컴파일러가 추론한 것과 어긋나는 경우
  - 참조 동일성 자체가 "성능"이 아니라 "정확성"을 위해 필요한 경우(예: `useEffect`의 의존성 배열이 매번 같은 객체 참조를 기대하는 코드)
- 주의할 점은 **bailout이 조용히(silently) 일어난다**는 것이다. 에러가 나지 않기 때문에, `eslint-plugin-react-hooks`(v6+에 `react-hooks/unsupported-syntax` 규칙이 포함됨)를 켜두지 않으면 특정 컴포넌트가 최적화에서 빠진 걸 눈치채지 못할 수 있다.
- 특정 함수를 의도적으로 최적화 대상에서 빼고 싶을 때는 함수 맨 위에 디렉티브를 둔다.

```jsx
function ThirdPartyWrapper() {
  "use no memo"; // 서드파티 훅이 부수효과를 갖고 있어 컴파일러가 잘못 최적화할 수 있음
  useThirdPartyHook();
  // ...
}
```

<br>

#### 6. 정리

---

- React Compiler는 "언제 메모이제이션할지"를 사람이 아니라 **빌드타임 정적 분석**이 결정하도록 옮긴 것이다. `React.memo`/`useMemo`/`useCallback`이 사라지는 게 아니라, 대부분의 경우 직접 쓸 필요가 없어지고 이펙트 의존성처럼 "참조 동일성이 정확성에 직결되는" 곳에서만 명시적으로 남겨두면 된다.
- 다만 이 최적화는 컴포넌트가 **Rules of React를 지킨다는 전제** 위에서만 안전하다. 전제가 깨지면 컴파일러는 조용히 bailout하므로, 마이그레이션 시에는 반드시 관련 eslint 규칙을 켜서 어디가 최적화되고 있고 어디가 빠졌는지 확인해야 한다.
- 결국 04/05에서 다룬 "어디에 memo를 붙일까"라는 고민 자체를 없애는 방향이지만, 그 고민이 "왜 이 렌더는 캐시되지 않는가"라는 디버깅으로 형태만 바뀐 셈이다.

Sources:

- [React Compiler – React](https://react.dev/learn/react-compiler)
- [React Compiler: Introduction](https://react.dev/learn/react-compiler/introduction)
- ["use no memo" directive – React](https://react.dev/reference/react-compiler/directives/use-no-memo)
- [Components and Hooks must be pure – React](https://react.dev/reference/rules/components-and-hooks-must-be-pure)
- [Rules of React](https://react.dev/reference/rules)
