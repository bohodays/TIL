### 01. useTransition을 통한 빠른 반응성으로 유저 경험 향상시키기

<br>

#### 1. useTransition이란?

---

- useTransition은 React 18에서 도입된 훅으로, 사용자 인터페이스(UI)의 응답성과 성능을 향상시키기 위한 기능이다.
  주로 사용자에게 바로 피드백을 주고, 무거운 상태 업데이트는 천천히 처리하고 싶을 때 사용한다.
- useTransition을 이용해서 작업의 `우선순위`를 변경할 수 있다.

<br>

#### 2. Concurrency와 Parallelism

---

- React의 useTransition은 Concurrency(동시성) 개념을 UI에 적용한 대표적인 기능이다. 하나의 흐름 안에서 여러 작업을 "동시에 다루는" 방식이기 때문에 **동시성(Concurrency)** 이라고 볼 수 있다.

| 항목              | **Concurrency (동시성)**                                          | **Parallelism (병렬성)**                                |
| ----------------- | ----------------------------------------------------------------- | ------------------------------------------------------- |
| **정의**          | 하나의 프로세서(또는 쓰레드)가 여러 작업을 **교대로** 처리하는 것 | 여러 작업을 **동시에** 실행하는 것 (여러 프로세서 사용) |
| **핵심 목적**     | 작업을 동시에 **다루는 느낌**을 주는 것 (스케줄링)                | 작업을 **동시에 실행**해서 처리 속도를 높이는 것        |
| **하드웨어 요구** | 하나의 CPU에서도 가능                                             | 멀티코어 CPU 필요                                       |
| **예시**          | 웹 서버가 여러 요청을 처리할 때, 하나씩 번갈아 응답               | 대규모 데이터 연산을 여러 CPU 코어로 나눠서 동시에 처리 |
| **비유**          | 한 사람이 여러 고객을 번갈아 응대하는 것                          | 여러 사람이 각자 고객을 동시에 응대하는 것              |

<br>

#### 3. useTransion 사용 예시

- 아래의 예시는 input창에 영어 알파벳을 입력하면 해당 영어 알파벳을 포함하는 단어들만 하이라이트해서 표시해주는 기능이다.
- useTransition을 사용하지 않은 경우

  ```javascript
  function FilterWords() {
    const [list, setList] = useState([]);
    useEffect(() => {
      fetch(targetURL)
        .then((r) => r.text())
        .then((r) => setList(r.split("\n").sort()));
    }, []);
    const [filter, setFilter] = useState("");
    const handleChange = ({ target: { value } }) => {
      setFilter(value);
    };
    return (
      <main>
        <label>
          Filter:
          <input type="search" value={filter} onChange={handleChange} />
        </label>
        <Words list={list} filter={filter} />
      </main>
    );
  }
  ```

  - input에 알파벳을 입력할 때마다 handleChange메서드가 실행되면서 input의 value와 Words 컴포넌트를 모두 변경한다. 입력과 필터링이 동시에 발생하고, 이로인해 반응성과 렌더링 속도가 느려서 사용자 경험이 감소한다. input의 입력 자체도 Words 컴포넌트의 렌더링으로 인해 입력이 부드럽지 않다.

- useTransition을 사용하는 경우

  ```javascript
  function FilterWords() {
    const [list, setList] = useState([]);
    useEffect(() => {
      fetch(targetURL)
        .then((r) => r.text())
        .then((r) => setList(r.split("\n").sort()));
    }, []);
    const [filter, setFilter] = useState("");
    const [deferedFilter, setDeferedFilter] = useState("");

    const [isPending, startTransition] = useTransition();

    const handleChange = ({ target: { value } }) => {
      setFilter(value);

      startTransition(() => {
        setDeferedFilter(value);
      });
    };
    return (
      <main>
        <label>
          Filter:
          <input type="search" value={filter} onChange={handleChange} />
        </label>
        {isPending ? (
          <p>Loading</p>
        ) : (
          <Words list={list} filter={deferedFilter} />
        )}
      </main>
    );
  }
  ```

  - handleChange 메서드 안에서 Words 컴포넌트에 렌더링할 필터링 단어들을 분리하고 useTransition을 적용하여 작업 우선순위를 낮춘다. 이를 통해 input의 입력은 즉시 반영되고, Words 컴포넌트의 필터링된 렌더링은 낮은 우선순위로 처리되어 **입력창의 응답성을 높이고 UX를 개선한다.** input 입력 자체도 부드럽게 수행할 수 있다.

<br>

#### 4. useTransition의 단점 및 주의점

---

| 항목                                    | 설명                                                                                                                          |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| 🔻 **불필요한 복잡성 증가**             | 상태가 하나 더 생기고, `startTransition`으로 감싸야 하므로 코드가 복잡해질 수 있음                                            |
| 🐌 **반응 지연 가능성**                 | "낮은 우선순위"로 처리되므로 시스템이 바쁠 경우, 업데이트가 늦게 반영될 수 있음                                               |
| 🧠 **잘못 사용 시 UX 혼란**             | transition 중인 값과 실제 입력값이 다르므로 사용자가 "입력했는데 결과가 안 바뀐다"는 느낌을 받을 수 있음                      |
| 📦 **모든 렌더링 비용을 줄이지는 않음** | transition 처리된다고 해서 내부 컴포넌트의 모든 성능 문제가 해결되는 건 아님. 별도 최적화(`useMemo`, `memo`)도 필요할 수 있음 |
| 🚫 **조건부 렌더링이 꼬일 수 있음**     | `isPending` 상태와 `deferred value` 간 동기화를 잘못하면 화면이 이상하게 깜빡이거나 로직이 꼬일 수 있음                       |

<br>

- 언제사용하면 좋을까?

  | 사용 권장 상황                     | 설명                                                |
  | ---------------------------------- | --------------------------------------------------- |
  | ✅ 대량 데이터 렌더링              | 리스트가 크고 필터/정렬이 느릴 때                   |
  | ✅ 실시간 입력 UX 유지             | 입력 필드 반응은 즉각적이고 결과는 천천히 보여줄 때 |
  | ✅ 부하가 큰 연산이 UI와 연동될 때 | 예: 검색 자동완성, 고비용 컴포넌트 렌더링 등        |

<br>

- 사용 안 해도 되는 상황
  | 사용 비권장 상황 | 이유 |
  | --------------------------- | ----------------------------- |
  | ❌ 상태 업데이트가 가볍고 즉각 반영돼야 할 때 | transition을 써도 큰 차이 없음 |
  | ❌ 값과 UI가 완전히 실시간으로 일치해야 할 때 | deferred value가 UX 혼란을 줄 수 있음 |

<br>

- 정리
  - useTransition은 UX 최적화 도구일 뿐 성능 만능 해결책은 아니다. "입력은 빠르게, 결과는 천천히" 같은 구조가 필요한 곳에만 선택적으로 적용하는 것이 좋다.
- useTransition 비유
  - 커피숍에서 바리스타에게 "먼저 커피부터 주시고, 베이글은 천천히 주세요"라고 요청하는 것과 같다.
