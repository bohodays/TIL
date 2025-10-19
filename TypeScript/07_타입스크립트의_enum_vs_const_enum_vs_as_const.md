### 07.타입스크립트의 enum vs const enum vs as const

---

- enum
  - **기본적이고 직관적이지만 무겁다.**
  - enum은 "열거형 타입(Enumerated Type)으로, 정수나 문자열 같은 리터럴 값들의 집합을 이름있는 식별자로 묶어주는 기능
  - TS에서만 존재하는 기능으로, 자바스크립트에서는 기존에 없던 개념으로 TS가 제공하는 확장 기능
  - ```typescript
    enum Direction {
      Up,
      Down,
    }
    ```
  - 위의 타입스크립트 코드를 JS로 컴파일하면 아래와 같음
    - ```javascript
      "use strict";
      var Direction;
      (function (Direction) {
        Direction[(Direction["Up"] = 0)] = "Up";
        Direction[(Direction["Down"] = 1)] = "Down";
      })(Direction || (Direction = {}));
      console.log(Direction.Up);
      ```
  - 런타임에 실제 객체가 생성됨
  - 디버깅이 쉽고 직관적이지만 `번들에 그대로 포함되어 사이즈가 커짐`
  - `트리쉐이킹도 잘 되지 않아. 사용하지 않아도 번들에 남을 수 있음`
    - 트리쉐이킹이란?
      - 자바스크립트 모듈은 함수나 객체들이 트리(계층 구조) 형태로 의존 관계를 갖고 있고, 트리를 흔들어(shake) 필요 없는 가지(dead code)를 떨어뜨리는 것처럼, 사용하지 않는 코드를 번들에서 제거한다는 의미
      - 트리쉐이킹이 중요한 이유
        - 번들 크기 감소
          - 사용하지 않는 코드가 제거되어 최종 파일이 가벼워짐
        - 로딩 속도 개선
          - 네트워크로 다운받는 JS 용량이 줄어듦
        - 유지보수 용이
          - 불필요한 코드가 줄어들어 깔끔한 빌드 결과

<br>

- const enum

  - **가장 가볍지만 제약이 있음**
  - const enum은 enum의 단점(코드량 증가, 트리쉐이킹 어려움)을 보완하기 위해 도입된 기능
  - ```typescript
    const enum Direction {
      Up,
      Down,
    }

    console.log(Direction.Up);
    ```

  - 위의 타입스크립트 코드를 JS로 컴파일하면 아래와 같음
    - ```javascript
      console.log(0);
      ```
  - Direction.Up이 값으로 `인라인` 치환되며 객체 자체가 없음

    - `인라인 치환이란?`

      - 변수를 런타임에 해석하지 않고 컴파일 시점에 값으로 바꿔버리는 것
      - ```javascript
        const PI = 3.14;
        console.log(PI);

        // 컴파일하면 결과는 아래와 같음.
        console.log(3.14);
        ```

      - 인라인 치환의 장점
        - 성능 향상
          - 런타임에 변수나 함수 호출을 하지 않아도 됨
        - 번들 크기 감소
          - 변수 선언 코드 자체가 사라짐
        - 최적화 용이
          - 정적 값이 명확해지므로 최적화 여지가 커짐

  - 사용하지 않는 값은 번들에 전혀 포함되지 않기 때문에 트리쉐이킹 효과가 탁월함
  - 하지만 런타임에 enum이 없기 때문에 디버깅이 불편하고, SWC/Babel 환경에서는 에러가 날 수 있음

<br>

- as const

  - **실무에서 많이 사용하는 방법**
  - TS 3.4에 도입된 const assertion 문법에서 파생된 표현
  - ```typescript
    export const Direction = {
      Up: "UP",
      Down: "DOWN",
    } as const;

    export type Direction = (typeof Direction)[keyof typeof Direction];
    ```

  - 위의 타입스크립트 코드를 JS로 컴파일하면 아래와 같음
    - ```javascript
      export const Direction = {
        Up: "UP",
        Down: "DOWN",
      };
      ```
  - 런타임 객체가 유지되어 디버깅 용이
  - 사용하지 않는 경우에는 번들러의 트리쉐이킹으로 제거할 수 있음
  - 호환성이 좋아 실무나 라이브러리 배포에도 자주 사용
  - `const enum`처럼 값이 완전히 인라이닝되진 않기 때문에, 객체로서 번들에 남을 가능성이 있음
    - 번들 크기 측면에서 `const enum`보다는 약간 불리
