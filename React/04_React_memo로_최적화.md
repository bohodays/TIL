### 04. React.memo로 최적화

<br>

#### 1. React.memo란?

---

- props가 바뀌지 않았다면, 컴포넌트를 다시 렌더링하지 않도록 메모이제이션하는 고차 컴포넌트(Higher-Order Component)이다.
  - 메모이제이션
    - 같은 입력에 대해 함수가 동일한 결과를 반환하도록 결과를 저장(cache)해두는 기법

<br>

#### 2. 언제 리렌더링이 발생하나?

---

- React에서는 부모 컴포넌트가 리렌더링되면, 자식 컴포넌트도 기본적으로 리렌더링된다. 하지만 자식의 props가 바뀌지 않았을 경우, 굳이 리렌더링할 필요가 없을 것이다. 이때 React.memo를 사용하면 **이전 props와 비교해서 값이 같으면 리렌더링을 건너뛰게 된다.**

<br>

#### 3. 사용예시

```javascript
const User = React.memo(
  ({ user }) => {
    console.log("User rendered:", user.name);
    return <div>{user.name}</div>;
  },
  (prevProps, nextProps) => {
    // user의 id가 같으면 리렌더링 하지 않음
    return prevProps.user.id === nextProps.user.id;
  }
);

function App() {
  const [count, setCount] = useState(0);
  const [user, setUser] = useState({ id: 1, name: "Alice" });

  const updateUser = () => {
    setUser({ id: 1, name: "Alice" }); // 객체는 새로 만들어짐
  };

  return (
    <div>
      <button onClick={() => setCount((c) => c + 1)}>+ count</button>
      <button onClick={updateUser}>Set same user</button>
      <User user={user} />
    </div>
  );
}
```

- setUser({ id: 1, name: "Alice" })는 새로운 객체지만 비교 함수 (prevProps, nextProps) => prev.user.id === next.user.id 가 true를 반환하므로 `렌더링이 생략됨` (콘솔에 출력 안 됨)
- memo의 두 번째 인자 callback함수의 prevProps, nextProps을 잘 활용해서 렌더링을 최적화할 수 있다.
