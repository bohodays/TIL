### 06. 대량의 데이터 렌더링 최적화 (Virtualized List)

<br>

#### 1. Virtualized List란?

---

- **보이는 항목만 실제로 렌더링**하고, **화면 밖에 있는 수천 개의 항목은 렌더링하지 않음**으로써 성능을 최적화하는 기법
- 다른 프레임워크에서는 기본 컴포넌트로 제공하기도 한다.
  - React Native
    - FlatList, VirtualizedList
  - Android
    - RecyclerView
  - iOS
    - UICollectionView

<br>

#### 2. 왜 필요한가?

---

```javascript
{
  items.map((item) => <ItemComponent key={item.id} {...item} />);
}
```

- 1만 개의 항목을 한 번에 렌더링하면 → **브라우저 성능 저하**
- 느려지고, 스크롤도 끊기고, 메모리도 많이 씀
- Virtualized List
  - 예를 들어 화면에 10개만 보인다면, 그 10개만 렌더링
  - 스크롤 위치에 따라 필요한 항목만 렌더링하고, 나머지는 제거
  - DOM 수를 최소화해서 렌더링 성능 개선

<br>

#### 3. 예시 코드

---

```javascript
import React, { useState, useEffect, useRef } from "react";

interface Base {
  id?: string | number;
}

interface Props<T> {
  itemCount: number;
  items: (T & Base)[];
  renderItem: (item: T, index: number) => React.ReactNode;
  keyExtractor?: (item: T) => string;
  itemSize: number;
}

function RecyclerView<T>({
  itemCount,
  items,
  renderItem,
  keyExtractor,
  itemSize,
}: Props<T>) {
  const [visibleStartIdx, setVisibleStartIdx] = useState(0);
  const [visibleEndIdx, setVisibleEndIdx] = useState(10); // For initial render, rendering 10 items.
  const containerRef = useRef < HTMLDivElement > null;

  const handleScroll = () => {
    if (containerRef.current) {
      const { scrollTop } = containerRef.current;

      const newStartIdx = Math.floor(scrollTop / itemSize);
      const visibleItemsCount = Math.ceil(window.innerHeight / itemSize) + 2; // Adding 2 for buffer

      setVisibleStartIdx(newStartIdx);
      setVisibleEndIdx(newStartIdx + visibleItemsCount);
    }
  };

  useEffect(() => {
    if (containerRef.current) {
      containerRef.current.addEventListener("scroll", handleScroll);

      return () => {
        if (containerRef.current) {
          containerRef.current.removeEventListener("scroll", handleScroll);
        }
      };
    }
  }, []);

  return (
    <div
      ref={containerRef}
      style={{ overflowY: "auto", height: "100vh", position: "relative" }}
    >
      <div
        style={{
          width: "500px",
          height: itemCount * itemSize + "px",
          position: "relative",
        }}
      >
        {/* Container to hold actual height */}
        {items.slice(visibleStartIdx, visibleEndIdx).map((item, index) => (
          <div
            key={keyExtractor ? keyExtractor(item) : item.id}
            style={{
              top: (visibleStartIdx + index) * itemSize,
              position: "absolute",
              width: "100%",
            }}
          >
            {renderItem(item, visibleStartIdx + index)}
          </div>
        ))}
      </div>
    </div>
  );
}

export default RecyclerView;
```

- 위의 코드는 visibleStartIdx와 visibleEndIdx를 통해 필요한 아이템들만 렌더링하고, 스크롤에 따라 visibleStartIdx와 visibleEndIdx 값을 변경하고 있다.

<br>

#### 4. 장단점

---

- 장점
  - 렌더링 성능 대폭 향상
  - 스크롤도 부드럽고 빠름
  - 특히 모바일, 저사양 기기에서 필수
- 단점
  - 동적 높이 아이템 처리 어려움
  - 스크롤 위치 계산 로직 필요
  - SEO에서 불리 (보이지 않는 항목은 DOM에 없음)
- 직접 구현하기보다는 react-window와 같은 라이브러리를 사용하는 것을 추천한다.
