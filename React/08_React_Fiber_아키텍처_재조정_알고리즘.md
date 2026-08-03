### 08. React Fiber 아키텍처 (재조정 알고리즘)

<br>

#### 1. Fiber 이전: Stack Reconciler의 한계

---

- React 15까지는 재조정(Reconciliation)을 **재귀 함수 호출 스택**으로 처리했다.
  - 부모 → 자식으로 내려가며 렌더링하고, 트리 전체를 다 처리할 때까지 중간에 멈출 수 없었다.
- 문제
  - 컴포넌트 트리가 깊고 크면, 한 번의 업데이트가 메인 스레드를 오래 점유한다.
  - 그 사이에 들어온 사용자 입력(클릭, 타이핑)이나 애니메이션 프레임이 밀려서 화면이 버벅인다(**Jank**).
  > 찰떡 비유: 식당 주방
  - 주방장이 한 번 주문을 받으면 처음부터 끝까지 그 요리를 완성할 때까지 다른 손님을 절대 쳐다보지 않는 것과 같다. 손님이 아무리 급해도 순서가 오기 전까지는 기다려야 한다.
- React 16부터는 이 재귀 스택을 **Fiber**라는 자료구조로 갈아엎어서, 렌더링 작업을 "언제든 멈췄다가 이어서 할 수 있는" 단위로 쪼갰다.

<br>

#### 2. Fiber란 무엇인가?

---

- Fiber는 컴포넌트 하나하나에 대응하는 **JS 객체**이자, "작업 단위(Unit of Work)"다. 공식 문서/RFC에서는 **가상 스택 프레임**이라고 표현한다.
  > "A Fiber is a JavaScript object that represents a unit of work to be done."
  > Fiber는 처리해야 할 작업 단위를 나타내는 JS 객체다.
- 실제 콜 스택 대신, React가 직접 관리하는 **연결 리스트(linked list)** 형태로 트리를 표현한다.
  - `child` : 첫 번째 자식
  - `sibling` : 다음 형제
  - `return` : 부모(작업이 끝나면 돌아갈 곳)
  - `alternate` : 짝이 되는 반대편 fiber (아래 3번 항목 참고)
  - `pendingProps` / `memoizedProps` : 새로 들어온 props와, 이전에 기록해둔 props
- 이 구조 덕분에 React는 재귀 호출 대신 **while 루프**로 트리를 순회할 수 있고, 루프이기 때문에 중간에 멈추고 나중에 다시 시작하는 게 가능해진다.

```javascript
// 개념적으로 단순화한 workLoop
function workLoop(deadline) {
  while (nextUnitOfWork && deadline.timeRemaining() > 1) {
    nextUnitOfWork = performUnitOfWork(nextUnitOfWork);
  }
  // 시간이 부족하면 여기서 멈추고, 브라우저에게 제어권을 돌려준다
  requestIdleCallback(workLoop);
}
```

<br>

#### 3. Double Buffering: current 트리와 work-in-progress 트리

---

- Fiber는 화면에 이미 그려진 **current 트리**와, 다음 업데이트를 계산 중인 **work-in-progress(WIP) 트리** 두 개를 동시에 들고 있는다.
- 이 둘은 `alternate` 필드로 서로를 가리키며, 계산이 끝나면 두 트리의 역할을 통째로 스왑한다.
  > 찰떡 비유: 게임 화면의 더블 버퍼링
  - 화면에 지금 보여주는 프레임(current)과 그 뒤에서 미리 그리고 있는 다음 프레임(work-in-progress)을 따로 두는 것과 똑같다. 다 그리기 전의 어중간한 상태를 화면에 노출하지 않고, 완성된 프레임만 한 번에 교체(swap)한다.
- 이 덕분에 렌더링 도중 작업이 취소되거나 중단돼도, 화면에는 항상 완결된 이전 상태만 보이고 깜빡임이나 반쪽짜리 UI가 노출되지 않는다.

<br>

#### 4. Render Phase vs Commit Phase

---

- React의 업데이트는 크게 두 단계로 나뉜다.
  - **Render Phase (중단 가능)**
    - `beginWork` → `completeWork`를 반복하며 WIP 트리를 순회하고, 무엇이 바뀌었는지 계산한다.
    - 이 단계는 부수효과가 없는 **순수 계산**이라, 언제든 멈추거나 버려도 안전하다. 더 급한 업데이트(예: 사용자 클릭)가 들어오면 하던 계산을 버리고 새 계산을 먼저 처리할 수도 있다.
    - 계산 결과는 fiber마다 "무엇을 바꿔야 하는지" 표시(effect)로 쌓인다.
  - **Commit Phase (동기, 중단 불가)**
    - Render Phase가 끝나고 나온 effect 목록을 실제 DOM에 한 번에 반영한다.
    - DOM 조작, `useLayoutEffect`, ref 연결처럼 **부수효과가 있는 작업**이라 중간에 멈추면 화면이 어중간한 상태로 남으므로, 이 단계는 절대 끊기지 않는다.
- 정리하면 "무엇을 바꿀지 계산하는 것(Render)"과 "실제로 바꾸는 것(Commit)"을 분리했기 때문에, 계산 부분만 쪼개서 중단·재개·우선순위 조정이 가능해진 것이다.

<br>

#### 5. 우선순위 스케줄링 (Lane 모델)

---

- Fiber 트리가 있다고 모든 업데이트가 똑같이 급한 것은 아니다. React는 업데이트마다 **우선순위(Lane)**를 매겨서, 급한 것부터 처리한다.
  - 사용자의 클릭·타이핑 같은 즉각적인 입력 → 높은 우선순위
  - `useTransition`, 데이터 페칭 결과 반영 같은 백그라운드성 업데이트 → 낮은 우선순위
  > 찰떡 비유: 응급실 트리아지(Triage)
  - 접수 순서대로 진료하는 게 아니라, 위급한 환자부터 먼저 본다. React도 들어온 순서가 아니라 "얼마나 급한 업데이트인가"를 기준으로 먼저 처리할 작업을 고른다.
- 이 우선순위 개념이 있기 때문에 `useTransition`으로 특정 업데이트를 "급하지 않다"고 표시하거나, Automatic Batching으로 여러 업데이트를 하나의 커밋으로 묶는 최적화가 가능해진다. 즉, Fiber는 이 저장소에서 다룬 `useTransition`([01](https://github.com/bohodays/TIL/blob/master/React/01_useTransition을_통한_빠른_반응성으로_유저_경험_향상시키기.md)), `Automatic Batching`([02](<https://github.com/bohodays/TIL/blob/master/React/02_Automatic_batching(React_18).md>))이 실제로 돌아가는 **엔진**에 해당한다.

<br>

#### 6. 정리

---

- Stack Reconciler → Fiber로 바뀌면서 얻은 것
  - 렌더링 작업을 잘게 쪼개 **중단·재개**가 가능해짐 (Incremental Rendering)
  - 업데이트에 **우선순위**를 매겨 급한 작업을 먼저 처리 가능 (Concurrent Rendering의 토대)
  - Render/Commit을 분리해, 계산 단계에서 안전하게 작업을 버리거나 재시작 가능
- React.memo, useMemo, useTransition 같은 최적화 API들은 결국 이 Fiber 아키텍처가 제공하는 "쪼갤 수 있고, 순서를 바꿀 수 있는 작업 단위" 위에서 동작하는 것이다.

Sources:

- [An in-depth overview of the React fiber reconciliation algorithm](https://blog.ag-grid.com/inside-fiber-an-in-depth-overview-of-the-new-reconciliation-algorithm-in-react/)
- [acdlite/react-fiber-architecture](https://github.com/acdlite/react-fiber-architecture)
- [A deep dive into React Fiber - LogRocket Blog](https://blog.logrocket.com/deep-dive-react-fiber/)
