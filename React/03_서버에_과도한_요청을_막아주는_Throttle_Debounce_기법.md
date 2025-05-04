### 03. 서버에 과도한 요청을 막아주는 Debounce, Throttle 기법

<br>

#### 1. Debounce 기법과 Throttle 기법

---

- Debounce 기법과 Throttle 기법은 React에서 `사용자 입력이나 이벤트 처리를 최적화하여 과도한 서버 요청을 방지`할 때 사용하는 기법으로 useDebounce와 useThrottle과 같이 hook 형태로 만들어서 사용한다.

<br>

#### 2. Debounce 기법

---

- **입력이 끝난 후 일정 시간이 지나면 실행**
  - 사용자가 타이핑을 멈추면 서버 요청
  - 입력이 계속되면 요청이 지연됨
  - 마지막 동작만 반영됨
- 예시) 사용자가 검색창에 글을 입력할 때, 입력이 끝나고 500ms 후에 검색 API 호출

```javascript
// 예시: 입력이 멈춘 후 500ms가 지나면 검색 실행
const debouncedValue = useDebounce(value, 500);

function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const handler = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(handler);
  }, [value, delay]);
  return debounced;
}
```

#### 3. Throttle 기법

---

- **일정 시간마다 실행**
  - 사용자가 아무리 자주 이벤트를 발생시켜도, 정해진 시간 간격마다 한 번만 실행됨
- 예시) 스크롤 이벤트가 100번 발생해도 300ms마다 한 번만 서버에 위치 전송

```javascript
// 예시: 스크롤 이벤트를 500ms마다 한 번씩 처리
const throttledValue = useThrottle(value, 500);

function useThrottle(value, delay) {
  const [throttled, setThrottled] = useState(value);
  const lastRan = useRef(Date.now());

  useEffect(() => {
    const handler = setTimeout(() => {
      if (Date.now() - lastRan.current >= delay) {
        setThrottled(value);
        lastRan.current = Date.now();
      }
    }, delay - (Date.now() - lastRan.current));

    return () => clearTimeout(handler);
  }, [value, delay]);

  return throttled;
}
```
