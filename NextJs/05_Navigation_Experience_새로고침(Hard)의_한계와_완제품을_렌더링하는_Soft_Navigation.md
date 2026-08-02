### 05. Navigation Experience 새로고침(Hard)의 한계와 완제품을 렌더링하는 Soft Navigation

<br>

#### 1. Hard Navigation의 한계: '철거 후 재건축'의 비효율

---

- Next.js 도입 전, 우리가 전통적인 `<a>` 태그를 사용했던 시절은 **Hard Navigation** 의 시대였습니다. 이것은 웹의 탄생과 함께한 가장 원시적인 방식이다.
  > 비유: 이사 (포크레인 철거)
- 기존 방식의 페이지 이동은 옆집으로 이사를 가기 위해 **살던 집을 포크레인으로 완전히 부수고, 새 땅에 기초 공사부터 다시 시작하는 것** 과 같다. 브라우저는 현재 페이지를 파괴(Unmount)하고, 서버로부터 새로운 HTML을 받아 처음부터 다시 그린다.
- 하드 네비게이션의 3대 페인 포인트 (Pain Points)
  - **화면 깜빡임 (White Flash)**: 집이 부서지고 새 집이 지어지는 찰나의 순간, 사용자는 하얀 공백 화면을 마주해야 한다.
  - **상태 소멸 (State Loss)**: 열심히 작성하던 댓글, 스크롤 위치, 열어둔 사이드바 등 메모리에 있던 모든 정보가 '새로고침'과 함께 증발한다.
  - **비효율**: 공통으로 쓰는 헤더나 푸터(Header/Footer)까지 모두 부수고 다시 그린다.

    ```jsx
    /* [Pain Point]: Hard Navigation (HTML 기본 태그) - 파괴적인 이동 */

    export default function NavBar() {
      return (
        <nav>
          {/* 클릭하는 순간 브라우저가 '새로고침' 되며 모든 상태가 초기화된다. */}
          {/* 마치 집을 부수고 다시 짓는 것과 같다. */}
          <a href="/dashboard">Dashboard (Hard)</a>
          <a href="/settings">Settings (Hard)</a>
        </nav>
      );
    }
    ```

<br>

#### 2. Solution: Soft Navigation과 '완제품 가구' 배송

---

- Next.js는 이 문제를 `<Link>` 컴포넌트를 통한 **Soft Navigation** 으로 해결했다.
- 핵심 개념: RSC Payload (부분 리모델링)
  - 일반적인 리액트(SPA)가 '목재와 나사(JSON)'만 보내줘서 브라우저가 직접 조립해야 했다면, Next.js는 공장에서 이미 조립된 '완제품 가구(RSC Payload)'를 보내준다. 브라우저는 "거실 벽지는 그대로 두고, 소파만 이걸로 바꿔!"라는 지시를 받고, 필요한 부분만 '톡' 하고 갈아 끼운다.

> 비유: React Router vs Next.js Router

- React Router: 텅 빈 방에 목재(JSON)와 설명서(JS)가 도착한다.. 브라우저가 땀 흘려 가구를 조립해야 화면이 뜬다. (클라이언트 연산 부담 ⬆️)
- Next.js Router:완성된 가구(RSC Payload)가 도착한다. 브라우저는 배달 온 가구를 제자리에 놓기만 하면 된다. (브라우저 부담 ⬇️, 속도 ⬆️)

```jsx
/* [Solution]: Next.js <Link> (Soft Navigation) - 부드러운 전환 */
import Link from "next/link";

export default function NavBar() {
  return (
    <nav>
      {/* 클릭 시 서버에서 '완제품(RSC Payload)'만 받아와 즉시 교체한다. */}
      {/* 새로고침이 발생하지 않아 스크롤, 영상 재생 상태가 유지된다. */}
      <Link href="/dashboard">Dashboard (Soft)</Link>
      <Link href="/settings">Settings (Soft)</Link>
    </nav>
  );
}
```

<br>

#### 3. Deep Dive: '프리패칭(Prefetching)'이라는 마법

---

- Next.js의 `<Link>`에는 또 하나의 숨겨진 무기가 있다. 바로 **미리 가져오기(Prefetching)** 이다.
- 동작 원리: 셰프의 예측 요리
  - 식당에서 메뉴판의 '파스타'를 뚫어져라 쳐다보고 있다고 상상해 보자. 눈치 빠른 셰프(Next.js)는 주문하기도 전에 주방에서 이미 면을 삶기 시작한다.
    - **감지 (Viewport)**: 링크가 사용자의 화면(눈)에 들어오는 순간, Next.js가 감지한다.
    - **다운로드 (Background)**: 사용자가 클릭도 하기 전에, 백그라운드에서 다음 페이지의 데이터를 몰래 받아온다.
    - **즉시 전환 (Instant Transition)**: 사용자가 실제로 클릭했을 때, 이미 요리는 끝났다. 네트워크 대기 시간 없이 0초 만에 화면이 전환된다.
