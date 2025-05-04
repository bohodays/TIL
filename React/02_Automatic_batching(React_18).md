### 02. Automatic batching (React 18)

<br>

#### 1. React 18버전의 Automatic batching이란?

---

- React 18의 Automatic Batching은 여러 상태 업데이트를 자동으로 `하나의 렌더링`으로 묶어주는 기능이다. 이로 인해 불필요한 재렌더링을 줄이고 성능을 향상시킬 수 있다.
- 아래와 같이 root 요소를 creatRoot로 감싸야 적용된다.

```javascript
import App from "./App";
import { createRoot } from "react-dom/client";
import { StrictMode } from "react";

const rootElement = document.getElementById("root");
const root = createRoot(rootElement);

root.render(
  <StrictMode>
    <App />
  </StrictMode>
);
```

<br>

#### 2. Automatic batching이 적용되지 않은 경우(React 18 이전 버전)

---

```javascript
import "./styles.css";
import React, { useEffect } from "react";

const getUsers = () =>
  new Promise((resolve) => setTimeout(() => resolve(), 1000));

export default function App() {
  const [count1, setCount1] = React.useState(0);
  const [count2, setCount2] = React.useState(0);

  useEffect(() => {
    console.log("count1", count1);
    console.log("count2", count2);
  }, [count1, count2]);

  const onClick = async () => {
    await getUsers();
    setCount1(count1 + 1);
    setCount2(count2 + 1);
  };

  return (
    <div className="App">
      <h1>Counter</h1>
      <h2>Count1: {count1}</h2>
      <h2>Count2: {count2}</h2>
      <button onClick={onClick}>Click</button>
    </div>
  );
}
```

- 위의 코드를 실행시키고 Click 버튼을 누르면 아래와 같이 console이 찍힌다.
  - count1 1
  - count2 0
  - count1 1
  - count2 1

<br>

#### 3. Automatic batching이 적용된 경우(React 18 버전)

---

```javascript
import "./styles.css";
import React, { useEffect, experimental_use as use } from "react";

const getUsers = () =>
  new Promise((resolve) => setTimeout(() => resolve(), 1000));

export default function App() {
  const [count1, setCount1] = React.useState(0);
  const [count2, setCount2] = React.useState(0);

  useEffect(() => {
    console.log("count1", count1);
    console.log("count2", count2);
  }, [count1, count2]);

  const onClick = async () => {
    await getUsers();
    setCount1(count1 + 1);
    setCount2(count2 + 1);
  };

  return (
    <div className="App">
      <h1>Counter</h1>
      <h2>Count1: {count1}</h2>
      <h2>Count2: {count2}</h2>
      <button onClick={onClick}>Click</button>
    </div>
  );
}
```

- 위의 코드를 실행시키고 Click 버튼을 누르면 아래와 같이 console이 찍힌다.
  - count1 1
  - count2 1
- Automatic batching을 통해 불필요한 렌더링을 감소시켜 성능을 최적화시킬수 있다.
