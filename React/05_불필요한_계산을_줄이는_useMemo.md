### 05. 불필요한 계산을 줄이는 useMemo

<br>

#### 1. useMemo란?

---

- 값을 메모이제이션(memoization)해서 불필요한 연산을 방지하는 React 훅이다.

```javascript
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

- a나 b가 변하지 않으면, computeExpensiveValue는 다시 실행되지 않음. 이전에 계산한 값을 기억(cache) 했다가 재사용함.

<br>

#### 2. 사용 목적

---

- 렌더링 최적화
  - 렌더링마다 반복되는 무거운 계산을 방지
  - 리렌더링 시에도 값이 불필요하게 다시 계산되는 것을 방지

<br>

#### 3. 예시

---

```javascript
function App({ items }) {
  const pokemonWithPower = useMemo(
    () =>
      pokemon
        .map((p) => ({
          ...p,
          power: calculatePower(p),
        }))
        .filter((p) => p.power >= threshold),
    [pokemon, threshold]
  );

  return <div>활성 항목 수: {pokemonWithPower}</div>;
}
```

- pokemon과 threshold가 변하지 않으면 계산을 반복하지 않음
- pokemon과 threshold가 변경될때만 재계산

<br>

#### 4. useMemo가 불필요한 경우

---

```javascript
const result = useMemo(() => a + b, [a, b]);
```

- 위와 같은 단순한 연산에는 useMemo를 쓰는 게 오히려 성능에 더 나쁠 수 있다. `메모이제이션 자체도 비용이 있기 때문이다.`
  - 메모이제이션은 값을 캐시에 저장하므로 메모리를 사용한다.
  - useMemo는 컴포넌트가 언마운트되기 전까지 이전 값을 계속 보관한다.
  - 너무 많이 쓰거나, 큰 데이터를 저장하면 → 메모리 사용량이 늘어날 수 있다.

<br>

#### 5. useMemo를 고려해야 하는 경우

---

- 큰 배열에서 특정 조건으로 필터링할 때
- 컴포넌트 렌더링마다 객체가 새로 만들어지는 걸 막고 싶을 때
- React.memo 자식 컴포넌트에 props로 객체를 줄 때 (참조 유지용)
