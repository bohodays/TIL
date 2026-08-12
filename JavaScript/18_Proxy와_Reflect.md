### 18. Proxy와 Reflect

<br>

#### 1. Proxy — 객체 동작을 가로채는 대리인

---

- `Proxy`는 객체의 기본 동작(속성 읽기, 쓰기, 삭제, 함수 호출 등)을 가로채서 커스텀 로직을 끼워 넣을 수 있게 해주는 객체다.
  > 찰떡 비유: 비서를 거쳐야 만날 수 있는 대표
- 원본 객체(target)에 직접 접근하는 대신, 항상 비서(Proxy)를 거친다. 비서는 "누가 대표(target)를 찾아왔다", "무엇을 바꾸려 한다" 같은 요청을 가로채서 미리 정해둔 규칙(handler)에 따라 처리하고, 규칙에 없는 요청은 그대로 대표에게 전달한다.

```javascript
const target = { name: "홍길동" };

const handler = {
  get(obj, prop) {
    console.log(`${prop} 속성을 조회함`);
    return obj[prop];
  },
};

const proxy = new Proxy(target, handler);
proxy.name; // "name 속성을 조회함" 출력 후 "홍길동" 반환
```

- `handler`에 정의하는 함수를 **트랩(trap)** 이라 부르는데, 각 트랩은 자바스크립트 엔진 내부 동작인 [[Get]], [[Set]] 같은 **내부 메서드(internal method)** 를 가로챈다.

| 내부 메서드 | 트랩 | 가로채는 동작 |
|---|---|---|
| `[[Get]]` | `get()` | 속성 접근 (`obj.a`) |
| `[[Set]]` | `set()` | 속성 할당 (`obj.a = 1`) |
| `[[HasProperty]]` | `has()` | `in` 연산자 |
| `[[Delete]]` | `deleteProperty()` | `delete` 연산자 |
| `[[Call]]` | `apply()` | 함수 호출 |
| `[[Construct]]` | `construct()` | `new` 연산자 |

<br>

#### 2. Reflect — Proxy와 항상 짝을 이루는 이유

---

- 트랩 안에서 "가로챈 다음 원래 동작도 그대로 수행"해야 하는 경우가 대부분이다. 이때 `obj[prop] = value` 같은 문법을 직접 쓰는 대신 `Reflect`를 쓰는 게 표준이다.

```javascript
const handler = {
  get(target, prop, receiver) {
    console.log(`Getting ${prop}`);
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value, receiver) {
    console.log(`Setting ${prop} = ${value}`);
    return Reflect.set(target, prop, value, receiver);
  },
};
```

- `Reflect`는 `Math`처럼 인스턴스를 만들 수 없는 정적 메서드 모음이며, 모든 트랩 종류에 1:1로 대응하는 메서드를 제공한다(`Reflect.get`, `Reflect.set`, `Reflect.has`, `Reflect.deleteProperty`, `Reflect.apply`, `Reflect.construct` ...). `delete target[prop]`처럼 트랩마다 다른 연산자·문법을 외울 필요 없이 일관된 함수 호출로 기본 동작을 재현할 수 있다.
- 여기서 `receiver`를 반드시 함께 넘겨야 하는 실무 엣지케이스가 하나 있다. `Reflect.get(target, prop)`까지만 쓰고 `receiver`를 생략하면, 프로토타입 체인을 타고 올라가서 찾은 getter의 `this`가 **target으로 고정**돼 버린다.

```javascript
const target = {
  _x: 10,
  get x() {
    return this._x;
  },
};

const proxy = new Proxy(target, {
  get(target, prop, receiver) {
    // ❌ receiver를 생략하면 getter의 this가 target으로 고정된다.
    // return Reflect.get(target, prop);

    // ✅ receiver를 그대로 넘겨야 proxy를 상속한 객체에서도 this가 올바르게 유지된다.
    return Reflect.get(target, prop, receiver);
  },
});

const child = Object.create(proxy);
child._x = 99;
child.x; // receiver 전달 시 99, 생략 시 target._x인 10
```

- 즉 `Reflect`는 "몰라도 되는 편의 기능"이 아니라, 프로토타입 상속과 얽힌 상황에서 Proxy가 **원본과 동일하게 동작**하도록 보장해주는 필수 짝이다.

<br>

#### 3. 실무 활용 사례

---

- **유효성 검사**: `set` 트랩에서 잘못된 값이 들어오면 즉시 예외를 던진다.

```javascript
const withValidation = new Proxy(
  {},
  {
    set(target, prop, value, receiver) {
      if (prop === "age") {
        if (!Number.isInteger(value)) {
          throw new TypeError("age는 정수여야 합니다.");
        }
        if (value < 0 || value > 150) {
          throw new RangeError("age 값이 유효 범위를 벗어났습니다.");
        }
      }
      return Reflect.set(target, prop, value, receiver);
    },
  },
);

withValidation.age = 30; // OK
withValidation.age = -5; // RangeError
```

- **읽기 전용(불변) 객체 흉내**: `set`/`deleteProperty`를 막아 실수로 상태를 변경하는 걸 방지한다. `Object.freeze`와 달리 "왜 막혔는지"를 로깅하거나 조건부로 막는 등 세밀한 제어가 가능하다.

```javascript
function readonly(target) {
  return new Proxy(target, {
    set() {
      console.warn("읽기 전용 객체는 수정할 수 없습니다.");
      return false; // false를 반환하면 strict mode에서 TypeError가 발생한다.
    },
    deleteProperty() {
      console.warn("읽기 전용 객체의 속성은 삭제할 수 없습니다.");
      return false;
    },
  });
}
```

- **프레임워크의 반응형(reactivity) 시스템**: Vue 3는 Vue 2가 쓰던 `Object.defineProperty` 기반 접근 대신 `Proxy`로 반응형 데이터를 구현한다. `defineProperty`는 객체를 순회하며 이미 존재하는 속성마다 getter/setter를 미리 심어둬야 해서 **나중에 추가되는 속성이나 배열 인덱스 변경을 감지하지 못하는 한계**가 있었다. `Proxy`는 객체 자체를 감싸기 때문에 속성이 언제 추가되든 `get`/`set` 트랩이 그대로 걸린다.

```javascript
function reactive(target) {
  return new Proxy(target, {
    get(target, prop, receiver) {
      track(target, prop); // 의존성 추적
      return Reflect.get(target, prop, receiver);
    },
    set(target, prop, value, receiver) {
      const result = Reflect.set(target, prop, value, receiver);
      trigger(target, prop); // 변경 알림 → 리렌더링
      return result;
    },
  });
}
```

<br>

#### 4. 알아둬야 할 제약 — Invariants와 성능

---

- Proxy 트랩은 자유롭게 아무 값이나 반환할 수 없다. 엔진이 강제하는 의미론적 규칙을 **불변식(invariants)** 이라 하는데, 위반하면 트랩이 아니라 `TypeError`가 발생한다.
  - target에 **non-configurable**로 정의된 데이터 속성은 `get` 트랩도 실제 값과 동일한 값을 반환해야 한다.
  - `set` 트랩은 성공 여부를 나타내는 boolean을 반환해야 한다.
  - target이 `Object.preventExtensions()`로 확장이 막혀 있으면, 트랩에서 새 속성을 추가한 것처럼 속일 수 없다.
- **private 클래스 필드는 Proxy를 통과하지 못한다.** private 필드 접근은 내부적으로 "정확히 이 클래스로 만들어진 인스턴스인가"를 검사하는데, Proxy는 target과 다른 별개의 객체이므로 이 검사를 통과하지 못해 `TypeError`가 난다.

```javascript
class Secret {
  #value;
  constructor(value) {
    this.#value = value;
  }
  reveal() {
    return this.#value;
  }
}

const secret = new Secret(42);
const proxy = new Proxy(secret, {});
proxy.reveal(); // TypeError: Cannot read private member #value ...
```

- **성능 오버헤드**: 트랩을 거치는 만큼 일반 객체 접근보다 느리다. 매 프레임 수천 번씩 접근하는 hot path(예: 렌더링 루프 내부 계산)에 무분별하게 Proxy를 씌우면 병목이 될 수 있으니, 반응형 데이터처럼 "가로챌 필요가 명확한 경계"에만 적용하는 게 안전하다.

<br>

#### 5. 정리

---

- `Proxy`는 객체의 기본 동작(get/set/delete/call/construct 등)을 가로채 커스텀 로직을 끼워 넣는 대리 객체다.
- `Reflect`는 트랩 안에서 원래 동작을 표준화된 방식으로 재현하기 위한 짝꿍이며, 특히 `get`/`set` 트랩에서 `receiver`를 함께 넘기지 않으면 프로토타입 체인을 통한 상속 시 `this`가 깨질 수 있다.
- 유효성 검사, 읽기 전용 객체, 로깅, 그리고 Vue 3의 반응형 시스템처럼 "속성 추가/변경을 실시간으로 감지해야 하는" 상황에서 `Object.defineProperty`의 한계를 넘어서는 대안으로 쓰인다.
- private 필드는 Proxy를 통과할 수 없고, 트랩을 거치는 만큼 오버헤드가 있으므로 hot path보다는 경계가 뚜렷한 지점에 적용하는 것이 좋다.
