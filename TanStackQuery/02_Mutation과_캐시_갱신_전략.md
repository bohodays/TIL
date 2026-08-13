### 02. Mutation과 캐시 갱신 전략

<br>

#### 1. Query와 Mutation은 왜 분리되어 있을까

---

- `useQuery`가 "읽기"를 담당한다면 `useMutation`은 "쓰기/부수효과"를 담당한다. 공식 문서는 이 구분을 이렇게 명시한다.
  > "Unlike queries, mutations are typically used to create/update/delete data or perform server side-effects."
  > 쿼리와 달리 뮤테이션은 보통 데이터를 생성·수정·삭제하거나 서버 사이드 이펙트를 일으키는 데 쓰인다.
- `useQuery`는 `queryKey`만 같으면 자동으로 캐싱·중복 제거·백그라운드 리페치가 일어나지만, `useMutation`은 그런 자동화가 없다 — POST/PUT/DELETE 같은 요청은 멱등하지 않아서 "같은 요청이니 재사용"이라는 캐싱 개념 자체가 성립하지 않기 때문이다. 대신 `useMutation`은 **요청이 끝난 뒤 캐시를 어떻게 갱신할지**에 초점이 맞춰진 API를 제공한다.

```jsx
const mutation = useMutation({
  mutationFn: (newTodo) => axios.post("/todos", newTodo),
});

mutation.mutate({ title: "TIL 쓰기" });
```

<br>

#### 2. 콜백 라이프사이클 — `onMutate` → `onSuccess`/`onError` → `onSettled`

---

- `useMutation`에는 네 가지 콜백이 있고, 실행 순서와 역할이 고정되어 있다.
  1. **`onMutate(variables)`** — `mutationFn`이 실제로 요청을 보내기 **직전**에 실행된다. 낙관적 업데이트를 여기서 시작한다.
  2. **`onSuccess`** 또는 **`onError`** — 요청 결과에 따라 둘 중 하나만 실행된다.
  3. **`onSettled`** — 성공/실패와 무관하게 **항상 마지막**에 실행된다.
- 헷갈리기 쉬운 지점 하나: `useMutation({...})`에 정의한 콜백은 `mutate()`를 호출할 때마다 매번 실행되지만, `mutate(variables, { onSuccess: ... })`처럼 **호출 시점에 넘긴 콜백은 그 호출 한 번에 대해서만** 실행된다. 컴포넌트 전역 로직(캐시 갱신 등)은 훅 정의부에, 그 화면에서만 필요한 후처리(모달 닫기, 토스트 띄우기 등)는 호출부에 넘기는 식으로 구분해서 쓰면 된다.
- v5부터 로딩 상태 값이 `isLoading`에서 **`isPending`**으로 바뀌었다. 예전 자료(v4 이하)를 참고할 때 자주 헷갈리는 지점이니 주의.

<br>

#### 3. 캐시 갱신 방법 두 가지 — `invalidateQueries` vs `setQueryData`

---

- **`invalidateQueries`**: 해당 `queryKey`를 stale로 마킹하고, 활성 구독이 있으면 즉시 백그라운드 리페치를 트리거한다. "서버가 진실의 원천"이라는 전제하에 다시 물어보는 방식이라 구현이 단순하고 안전하지만, 네트워크 왕복 시간만큼 화면 반영이 늦다.
- **`setQueryData`**: 네트워크 요청 없이 캐시를 직접 덮어쓴다. mutation 응답으로 받은 `data`를 그대로 캐시에 반영하거나, 예상되는 결과를 미리 그려 넣을 때(낙관적 업데이트) 쓴다. 같은 `queryKey`를 구독하는 화면이 여러 곳이어도 한 번의 호출로 전부 동기화된다.

```jsx
// 방법 A: 서버를 다시 믿고 물어본다 — 단순하지만 한 박자 느리다
onSuccess: () => queryClient.invalidateQueries({ queryKey: ["todos"] });

// 방법 B: 응답으로 받은 값을 캐시에 바로 꽂는다 — 즉시 반영되지만 API 응답 형태에 의존한다
onSuccess: (data) => queryClient.setQueryData(["todos", data.id], data);
```

- 실무에서는 대개 둘을 같이 쓴다: `onMutate`에서 `setQueryData`로 화면을 즉시 낙관적으로 갱신하고, `onSettled`에서 `invalidateQueries`로 서버의 최종 진실값과 다시 맞춘다.

<br>

#### 4. Optimistic Update — 세이브 포인트를 찍고, 실패하면 그 지점으로 되돌린다

---

- 낙관적 업데이트는 서버 응답을 기다리지 않고 성공을 가정한 채 화면을 먼저 바꾼 뒤, 실패하면 원래 상태로 되돌리는 패턴이다. 공식 가이드가 제시하는 표준 4단계는 게임의 세이브/로드와 닮아 있다 — 위험한 행동(요청)을 하기 전에 세이브 포인트(스냅샷)를 찍어두고, 실패하면 그 지점으로 로드(롤백)한다.

```jsx
useMutation({
  mutationFn: updateTodo,
  onMutate: async (newTodo) => {
    // 1. 진행 중인 리페치를 취소한다 (안 하면 낙관적 업데이트가 뒤늦게 덮어씌워질 수 있다)
    await queryClient.cancelQueries({ queryKey: ["todos"] });

    // 2. 롤백에 쓸 이전 값을 스냅샷으로 저장한다
    const previousTodos = queryClient.getQueryData(["todos"]);

    // 3. 캐시를 낙관적으로 미리 갱신한다
    queryClient.setQueryData(["todos"], (old) => [...old, newTodo]);

    // 4. 스냅샷을 context로 반환한다 — onError에서 이 값을 받는다
    return { previousTodos };
  },
  onError: (err, newTodo, context) => {
    // 실패하면 세이브 포인트로 되돌린다
    queryClient.setQueryData(["todos"], context.previousTodos);
  },
  onSettled: () => {
    // 성공/실패 무관하게 서버 값과 최종 동기화
    queryClient.invalidateQueries({ queryKey: ["todos"] });
  },
});
```

- 여러 화면이 같은 데이터를 구독하지 않고, mutation이 화면 한 곳에만 영향을 준다면 캐시를 직접 만지지 않고 `mutation.variables`와 `isPending`만으로 임시 UI를 그리는 더 단순한 방식도 있다. 이 경우 롤백 로직 자체가 필요 없다.

```jsx
const { isPending, variables, mutate, isError } = useMutation({
  mutationFn: (newTodo) => axios.post("/api/todos", { text: newTodo }),
  onSettled: () => queryClient.invalidateQueries({ queryKey: ["todos"] }),
});

// 실제 목록 + pending 중인 항목을 반투명하게 얹어서 보여준다
{
  isPending && <li style={{ opacity: 0.5 }}>{variables}</li>;
}
{
  isError && (
    <li style={{ color: "red" }}>
      {variables}
      <button onClick={() => mutate(variables)}>재시도</button>
    </li>
  );
}
```

<br>

#### 5. 흔한 함정 — `cancelQueries`를 빼먹으면 생기는 race condition

---

- `onMutate`에서 `cancelQueries`를 호출하지 않으면, 낙관적으로 캐시를 덮어쓴 직후 **이미 진행 중이던 백그라운드 리페치**가 뒤늦게 도착해서 방금 그린 낙관적 값을 다시 덮어써버릴 수 있다. 사용자 입장에서는 "입력한 게 잠깐 반영됐다가 갑자기 원래 값으로 되돌아간다"는 버그로 보인다. 캐시를 직접 만지기 전에는 항상 해당 `queryKey`의 진행 중인 요청부터 끊어야 한다.
- context 스냅샷(`previousTodos`)을 저장하지 않고 `onError`에서 롤백을 시도하면, 되돌릴 "이전 값" 자체를 알 수 없다. 스냅샷은 선택이 아니라 롤백의 전제 조건이다.
- `onSettled`의 `invalidateQueries`를 생략하면, 서버가 응답에 추가 필드(생성된 `id`, `createdAt` 등)를 채워 보내는 경우 낙관적으로 그렸던 값과 실제 서버 값이 영구히 어긋난 채로 캐시에 남을 수 있다.

<br>

#### 6. 그 밖의 API — `mutationKey`, `scope`, `useMutationState`

---

- **`mutationKey`**: mutation을 식별하는 키. 오프라인 지속성(persist/resume)이나 `useMutationState`로 특정 mutation의 진행 상태를 다른 컴포넌트에서 조회할 때 필요하다.
- **`scope`**: `scope: { id: "todo" }`처럼 지정하면 같은 `scope.id`를 가진 mutation들이 동시에 실행되지 않고 **순차적으로** 실행된다. 같은 리소스에 대한 연속 요청의 순서 보장이 필요할 때 유용하다.
- **`useMutationState`**: 특정 컴포넌트 밖에서 발생한 mutation의 상태나 variables를 구독하는 훅. 예를 들어 목록 화면이 아닌 다른 컴포넌트(헤더의 저장 인디케이터 등)에서 현재 진행 중인 mutation이 있는지 알아야 할 때 쓴다.

```jsx
const pendingTodos = useMutationState({
  filters: { mutationKey: ["addTodo"], status: "pending" },
  select: (mutation) => mutation.state.variables,
});
```
