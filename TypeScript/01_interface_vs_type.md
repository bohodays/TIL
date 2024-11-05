### 01. interface vs type

---

- interface

  - 객체의 구조를 정의할 때 주로 사용
  - 클래스나 객체가 따라야 할 규칙(규약)을 명시하는데 유용
  - 확장(extends)과 규약(implements) 가능

  ```typescript
  interface Person {
    name: string;
    age: number;
  }

  const person: Person = {
    name: "Alice",
    age: 25,
  };
  ```

<br>

- type

  - 더 일반적이고, 복잡한 타입을 정의할 때 사용
  - 객체, 함수, 배열 등 다양한 타입을 정의할 수 있으며, Union(조합), Intersection(교차) 타입 지원

  ```typescript
  type Person = {
    name: string;
    age: number;
  };

  const person: Person = {
    name: "Alice",
    age: 25,
  };
  ```

  ```typescript
  type AddFunction = (a: number, b: number) => number;

  const add: AddFunction = (a, b) => a + b;
  ```

<br>

- 확장성

  - interface

    - interface는 상속을 통한 확장과 다중 선언을 지원함.
    - 상속을 통한 확장

    ```typescript
    interface Animal {
      name: string;
    }

    interface Dog extends Animal {
      breed: string;
    }

    interface Cat extends Animal {
      color: string;
    }
    ```

    - 다중 선언 (같은 이름의 interface가 여러 번 선언되면 자동으로 병합)

    ```typescript
    interface Person {
      name: string;
      age: number;
    }

    interface Person {
      gender: string;
    }

    const person: Person = {
      name: "John",
      age: 30,
      gender: "Male",
    };
    ```

  - type

    - Union과 Intersection 타입을 통한 확장

    ```typescript
    type Animal = { name: string };
    type Dog = Animal & { breed: string }; // Intersection: Animal과 Dog 타입 결합
    type Cat = Animal & { color: string }; // Intersection: Animal과 Cat 타입 결합

    type Pet = Dog | Cat; // Union: Dog 또는 Cat 타입
    ```

    - type은 다중 선언(병합)을 지원하지 않음

    <br>

- 클래스에서의 사용

  - interface는 클래스가 구현해야 하는 규약을 정의하는데 사용 가능 (implements)

  ```typescript
  interface Animal {
    name: string;
    speak(): void;
  }

  class Dog implements Animal {
    name: string;
    constructor(name: string) {
      this.name = name;
    }
    speak() {
      console.log("Woof!");
    }
  }
  ```
