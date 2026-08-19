### 10. Suspense의 동작 원리와 스트리밍 SSR

<br>

#### 1. Suspense란 — "예외를 던져서 렌더링을 멈춘다"

---

- Suspense는 자식 컴포넌트가 아직 준비되지 않았을 때, 그 컴포넌트 대신 `fallback`을 보여주고 준비되면 자동으로 교체해주는 경계(boundary)다.
- 내부적으로는 **에러 처리와 똑같은 채널**을 재사용한다. 컴포넌트가 렌더링 도중 일반 에러 대신 **아직 resolve되지 않은 Promise를 throw**하면, React는 이걸 에러가 아니라 "잠깐 멈춰달라"는 신호로 해석한다.
  1. 컴포넌트 렌더링 중 Promise가 throw됨
  2. 가장 가까운 `<Suspense>` 경계가 이 Promise를 catch
  3. 그 경계는 자식 대신 `fallback`을 렌더링
  4. Promise가 resolve되면 React가 해당 부분을 다시 렌더링 시도

```jsx
<Suspense fallback={<Loading />}>
  <Albums artistId="the-beatles" />
</Suspense>
```

> 찰떡 비유: 식당의 "잠시만요" 팻말
>
> 손님(컴포넌트)이 주문한 요리가 아직 안 나왔을 때 테이블에 에러 카드를 놓는 게 아니라 "곧 나옵니다" 팻말(fallback)을 세워두는 것과 같다. 요리가 나오면 팻말을 치우고 요리를 그대로 올려준다 — 손님이 다시 주문할 필요는 없다.

<br>

#### 2. use()와 Promise 캐싱 — 왜 렌더 단계에서 던져야 하는가

---

- Suspense가 감지할 수 있는 건 **렌더링 함수가 실행되는 동안** 동기적으로 throw된 Promise뿐이다. `useEffect`나 이벤트 핸들러 안에서 비동기로 데이터를 가져오는 건 렌더 단계 밖의 일이라 Suspense가 아예 알아채지 못한다.

```jsx
// ❌ Effect에서 페칭 - Suspense가 감지 못함
function EffectAlbums({ artistId }) {
  const [albums, setAlbums] = useState([]);
  useEffect(() => {
    fetchData(`/${artistId}/albums`).then(setAlbums);
  }, [artistId]);
  return <ul>{albums.map((a) => <li key={a.id}>{a.title}</li>)}</ul>;
}

// ✅ 렌더 중 use()로 Promise를 읽음 - Suspense가 감지함
const cache = new Map();
function fetchData(url) {
  if (!cache.has(url)) cache.set(url, getData(url)); // 같은 Promise 인스턴스를 재사용
  return cache.get(url);
}

function Albums({ artistId }) {
  const albums = use(fetchData(`/${artistId}/albums`)); // 미해결 Promise면 여기서 throw됨
  return <ul>{albums.map((a) => <li key={a.id}>{a.title}</li>)}</ul>;
}
```

- 핵심은 **같은 요청에 대해 항상 같은 Promise 인스턴스를 반환하도록 캐싱**하는 것 — 매 렌더마다 새 Promise를 만들어 던지면 React는 "또 다른 작업이 시작됐다"고 오해해서 무한히 재시도만 반복하게 된다.
- 라이브러리(TanStack Query, SWR, Relay 등)가 자체 캐시를 갖고 있는 이유도 이 지점과 맞닿아 있다 — Suspense와 맞물리려면 요청 결과를 캐싱해서 "안정된 Promise"로 돌려줄 책임이 필요하기 때문이다.

<br>

#### 3. Fiber와의 연결 — Render Phase에서만 가능한 이유

---

- 08번 글에서 정리했듯 React의 업데이트는 [중단 가능한 Render Phase와 중단 불가능한 Commit Phase](https://github.com/bohodays/TIL/blob/master/React/08_React_Fiber_아키텍처_재조정_알고리즘.md#4-render-phase-vs-commit-phase)로 나뉜다.
- Promise throw는 오직 Render Phase(`beginWork`)에서만 의미가 있다 — 이 단계는 순수 계산이라 "이 fiber는 아직 완성 못 함, 나중에 다시" 하고 작업을 통째로 버리거나 미뤄도 안전하기 때문이다.
- Suspense가 fallback을 보여줄 때, 이미 화면에 있던 내용(current 트리)은 그대로 유지하고, 새로 계산 중이던 work-in-progress 쪽만 "미완성" 표시를 남긴 채 버려진다 — 08번에서 다룬 더블 버퍼링 구조 덕분에 화면에는 항상 완결된 상태만 노출된다.
- 즉 Suspense는 새로운 별도 기능이 아니라, **Fiber가 이미 갖고 있던 "작업을 중단하고 나중에 재개할 수 있는 능력"을 데이터 로딩에 활용한 것**에 가깝다.

<br>

#### 4. 재시도(retry)와 상태 초기화

---

- Promise가 resolve되면 React는 캐시된 결과를 반환하도록 Suspense 경계 아래를 **처음부터 다시 렌더링**한다. 이전에 진행되던 계산 결과를 이어 붙이는 게 아니라, mount가 완료되지 않은 컴포넌트의 로컬 state는 보존되지 않는다는 뜻이다.

```jsx
const [page, setPage] = useState("/");

function navigate(url) {
  startTransition(() => {
    setPage(url); // 상태가 바뀌면 관련 Suspense 트리가 다시 렌더링을 시도한다
  });
}
```

- 페이지 이동처럼 `startTransition`과 함께 쓰면, 이미 화면에 보이던 이전 콘텐츠는 그대로 둔 채 백그라운드에서 다음 화면을 준비하다가, 새 Suspense 경계들이 한꺼번에 준비됐을 때 화면을 교체한다 — 01번에서 다룬 `useTransition`의 "급하지 않은 업데이트" 개념이 여기서도 그대로 적용된다.

<br>

#### 5. 중첩된 Suspense — fallback은 각자, 완료는 독립적으로

---

```jsx
<Suspense fallback={<BigSpinner />}>
  <Biography artistId={artist.id} />
  <Suspense fallback={<AlbumsGlimmer />}>
    <Albums artistId={artist.id} />
  </Suspense>
</Suspense>
```

- 안쪽 경계가 따로 있으면, `Albums`가 아직 로딩 중이어도 `Biography`가 준비되는 순간 바깥쪽 `BigSpinner`는 먼저 걷히고 실제 콘텐츠가 보인다 — 안쪽의 `AlbumsGlimmer`만 남아서 부분적으로 로딩 표시를 이어간다.
- 즉 Suspense 경계는 **가장 가까운 조상 하나만** 책임지며, 여러 개를 겹쳐두면 "전체가 끝날 때까지 통째로 기다리기"가 아니라 **완료된 부분부터 순차적으로 드러내는** 형태로 UX를 세밀하게 조절할 수 있다.

<br>

#### 6. 스트리밍 SSR과 Selective Hydration

---

- 서버에서 `renderToPipeableStream`(Node) / `renderToReadableStream`(엣지)으로 렌더링하면, Suspense 경계를 만난 시점에 그 안쪽은 fallback으로 채운 채로 **먼저 완성된 HTML부터 스트리밍으로 클라이언트에 전송**한다.
- 서버에서 느린 데이터(예: 추천 상품 목록)가 나중에 준비되면, React는 스트림 뒤쪽에 `<script>` 조각을 추가로 흘려보내 fallback 자리를 실제 콘텐츠로 **교체**한다 — 페이지 전체가 완성될 때까지 기다리지 않고, 느린 부분만 늦게 채워 넣는 방식이다.
- 클라이언트 하이드레이션도 트리 전체를 한 번에 처리하지 않는다. React 18부터는 **Selective Hydration**이 적용되어, 아직 하이드레이션되지 않은 영역이라도 사용자가 그 영역을 클릭하면 React가 그 부분의 하이드레이션 우선순위를 즉시 끌어올린다.

> 찰떡 비유: 뷔페 오픈
>
> 모든 요리가 완성될 때까지 홀 문을 닫아두는 대신, 준비된 요리부터 순서대로 내놓고 나머지는 계속 조리하는 것과 같다(스트리밍). 손님이 특정 코너로 몰리면 그 코너의 요리사를 먼저 투입하는 것이 Selective Hydration에 해당한다 — 순서대로 처리하는 대신 "지금 필요한 곳"을 먼저 살아있게 만든다.

- 결과적으로 07번에서 살펴본 [RSC RFC](https://github.com/bohodays/TIL/blob/master/React/07_RFC_읽어보기_(Server_Components).md)가 그리던 "서버에서 스트리밍으로 점진적 렌더링"이라는 그림을, 실제로 가능하게 만드는 하부 메커니즘이 바로 이 Suspense + Fiber 조합이다.

<br>

#### 7. 정리

---

- Suspense는 새 렌더링 엔진이 아니라, **Fiber의 render/commit 분리 구조를 데이터 로딩에 맞게 활용한 코디네이션 primitive**다.
- 렌더 단계에서 캐싱된 Promise를 던져야 감지되고, resolve되면 처음부터 재시도하며, 중첩 경계는 각자 독립적으로 완료를 드러낸다.
- 스트리밍 SSR + Selective Hydration은 이 매커니즘을 서버 렌더링과 하이드레이션까지 확장한 것으로, "느린 부분 때문에 전체가 늦어지는" 구조를 없애는 것이 핵심이다.

Sources:

- [Suspense – React](https://react.dev/reference/react/Suspense)
- [Selective Hydration – patterns.dev](https://www.patterns.dev/react/react-selective-hydration/)
- [Streaming Server-Side Rendering – patterns.dev](https://www.patterns.dev/react/streaming-ssr/)
