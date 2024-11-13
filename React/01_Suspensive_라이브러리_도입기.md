### 01. Suspensive 라이브러리 도입기

---

- 사이드 프로젝트를 진행하면서 팀원의 추천으로 프로젝트에 @suspensive/react 라이브러리를 적용하기로 했습니다. 그래서 해당 라이브러리에 대한 설명과 도입기를 작성하게 되었습니다.
- 작성 내용은 공식 홈페이지를 참고했습니다.
  - https://suspensive.org/ko

<br>
<br>

0. Suspense와 ErrorBoundary 간단 설명

- Suspense는 비동기 작업의 로딩 상태를 쉽게 처리하게 도와줍니다.

  - React.lazy()와 함께 사용하여 컴포넌트를 동적으로(import) 로드할 때 사용됩니다.
  - Suspense는 fallback prop을 사용해 로딩 중 표시할 UI를 지정할 수 있습니다.
  - ```jsx
    import React, { Suspense } from "react";

    const LazyComponent = React.lazy(() => import("./LazyComponent"));

    function App() {
      return (
        <div>
          <h1>My App</h1>
          {/* Suspense로 감싸서 로딩 중일 때 표시할 UI를 지정 */}
          <Suspense fallback={<div>Loading...</div>}>
            <LazyComponent />
          </Suspense>
        </div>
      );
    }

    export default App;
    ```

  - React Query와 함께 사용하여 서버에 요청을 보내고 응답받는 동안 로딩 UI를 지정할 수 있습니다.
  - ```jsx
    import React, { Suspense } from "react";
    import { useQuery } from "react-query";
    import axios from "axios";

    function fetchData() {
      return axios.get("/api/data").then((res) => res.data);
    }

    function DataComponent() {
      const { data } = useQuery("data", fetchData, {
        suspense: true,
      });

      return <div>Data: {JSON.stringify(data)}</div>;
    }

    function App() {
      return (
        <Suspense fallback={<div>Loading...</div>}>
          <DataComponent />
        </Suspense>
      );
    }

    export default App;
    ```

- ErrorBoundary는 컴포넌트 트리의 에러를 감지하고 UI를 안전하게 렌더링할 수 있게 합니다.
  - 사용자 입력이나 외부 API 호출 등의 예상치 못한 에러 처리에 활용할 수 있습니다.
  - 네트워크 요청 실패 시 에러 핸들링을 할 수 있습니다.

<br>
<br>

1. @suspensive/react를 왜 사용해야 되는가?

- React Suspense는 때에 따라 서버사이드 렌더링을 피할 필요가 있습니다.
  - React Suspense는 NextJs와 같은 서버사이드 렌더링 환경에서 종종 문제가 발생합니다.
    - 대표적인 오류 - Hydration 오류
      - 서버에서 렌더링된 HTML과 클라이언트에서 생성된 React 트리가 일치하지 않을 때 발생
      - 서버에서는 데이터를 즉시 가져올 수 없는 경우, Suspense의 fallback UI가 렌더링됩니다. 하지만 클라이언트에서는 데이터가 준비되면 즉시 컴포넌트를 렌더링하게 되어, 서버와 클라이언트의 UI가 일치하지 않게 됩니다.
      - NextJs의 useClient, Server Components 활용, getServerSideProps을 통한 데이터 fetching 등 해결방법은 존재합니다.
  - @suspensive/react의 `<Suspense/>`에 clientOnly를 활용하면 해당 문제를 간단하게 해결할 수 있습니다.

<br>

- ErrorBoundary를 더욱 단순하게 사용하고 싶습니다.
  - React에서 ErrorBoundary를 사용하려면 직접 구현하거나 다른 라이브러리를 사용해야 합니다. @suspensive/react의 `<ErrorBoundary/>`를 사용하면 다소 복잡한 interface없이 사용 가능합니다.

<br>

- `<ErrorBoundary/>`의 fallback 외부에서 다수의 `<ErrorBoundary/>`를 reset하고 싶습니다.
  - `<ErrorBoundary/>`를 reset하려면 `<ErrorBoundary/>`의 fallback에 주어지는 reset을 사용하면 됩니다. 그러나 fallback 외부에서 다수의 `<ErrorBoundary/>`을 reset하려면 각 `<ErrorBoundary/>`의 props인 resetKeys에 새 resetKey를 제공해야 합니다. 하지만 @suspensive/react가 제공하는 `<ErrorBoundaryGroup/>`을 사용하면 이렇게 번거롭게 reset할 필요가 없습니다. `<ErrorBoundaryGroup/>`은 여러 `<ErrorBoundary/>`를 쉽게 재설정합니다.

<br>
- 결론 : React로 진행하는 사이드 프로젝트에서 서버사이드 렌더링과 에러 핸들링을 간단하게 하기 위해서 사용

<br>
<br>

2. 도입기

- @suspensive/react 라이브러리에서 제공하는 컴포넌트들의 사용법은 공식 홈페이지 참고바랍니다.
