### 07. RFC 읽어보기 (Server Components)

---

- RFC(Request for Comments)란?
  - 특정 프로젝트나 기술의 중요한 변경 사항이나 새로운 기능을 제안하고 논의하기 위한 공식적인 절차를 의미한다. 이는 단순한 아이디어 제안을 넘어서, 커뮤니티 전체가 함께 검토하고 합의하는 과정을 포함한다.
- RFC를 통해 리액트가 어떠한 과정을 통해 Server Components를 설계하고 도입하게 되었는지 이해하고, 그들의 고민과정을 살펴보고자 한다.

<br>

#### 1. Summary

---

- React의 Server Components는 서버와 클라이언트를 아우르는 기능으로, 클라이언트 앱의 풍부한 인터랙션과 전통적인 서버 렌더링의 성능 이점을 결합하는 것을 목표로 한다.

- 주요 내용
  - 서버 전용 실행
    - Server Components는 서버에서만 실행되며, 클라이언트 번들에 포함되지 않아 **번들 크기에 영향을 주지 않는다.**
  - 서버 데이터 접근
    - 데이터베이스, 파일 시스템, 마이크로서비스 등 서버 측 데이터 소스에 접근할 수 있다.
  - Client Components와의 통합
    - 서버에서 데이터를 로드해 Client Components에 props로 전달함으로써, 클라이언트는 인터랙티브한 부분만 렌더링한다.
  - 동적 렌더링
    - Server Components는 어떤 Client Components를 렌더링할지 동적으로 선택해, 클라이언트가 필요한 최소 코드만 다운로드하도록 한다.
  - 상태 보존
    - 페이지를 다시 불러오더라도 클라이언트 상태(포커스, 애니메이션 등)가 유지된다.
  - 점진적 렌더링
    - UI를 점진적으로 스트리밍하며, Suspense와 함께 사용해 의도적인 로딩 상태를 만들고 중요한 콘텐츠를 먼저 표시할 수 있다.
  - 코드 공유
    - 서버와 클라이언트 간에 동일한 컴포넌트를 재사용할 수 있어, 서버에서는 정적 콘텐츠를, 클라이언트에서는 편집 가능한 UI를 렌더링하는 식의 활용이 가능하다.

<br>

#### 2. Motivation

---

- Server Components는 다양한 React 앱에서 공통적으로 나타난 여러 문제를 해결하기 위해 고안되었다. 초기에는 이러한 문제들에 대해 부분적인 해결책을 찾는 단순한 접근을 시도했지만 그 과정에서 해당 접근 방식들에 만족하지 못했다. 근본적인 문제는 React앱이 **클라이언트 중심 구조**였고, 서버의 장점을 충분히 활용하지 못했다는 점이었다. 만약 개발자들이 서버를 더 쉽게 활용할 수 있게 된다면, 여러 문제들을 모두 해결하고 규모에 상관없이 더 강력한 앱 개발 방식을 제공할 수 있을 것이라고 판단했다.
- 이러한 문제들은 크게 2가지 범주로 나뉜다.
  - 개발자가 별도의 노력을 들이지 않아도 기본적으로 좋은 성능을 달성할 수 있도록 만드는 것
  - React앱에서 데이터를 가져오는 fetch 작업을 더 쉽게 만드는 것

<br>

**1) 번들 크기 문제 해결: Zero-Bundle-Size Components**

- 문제
  - React 앱은 대부분 클라이언트 중심으로 동작함.
  - 마크다운 렌더링, 날짜 포맷팅 등 편의를 위해 서드파티 라이브러리를 많이 사용하지만, 이는 번들 크기가 증가하고, 성능 저하로 이어짐.
    - 예: 마크다운 렌더링만으로도 240KB 이상의 JS가 필요할 수 있음.
- 해결
  - Server Component는 서버에서 실행되므로, 해당 코드가 클라이언트 번들에 포함되지 않아 번들 크기가 0이 됨.
  - Server Component에서 사용하는 라이브러리는 서버에서만 실행되므로, 클라이언트 번들에는 포함되지 않음.
- 예시

  ```JSX
    // Server Component
    import marked from 'marked';
    import sanitizeHtml from 'sanitize-html';

    export default function NoteWithMarkdown({ text })
    const html = sanitizeHtml(marked(text));
    return <div dangerouslySetInnerHTML={{ __html: html }} />;
  ```

- 효과
  - 번들 크기 감소
  - 로딩 속도 향상
  - 개발자는 동일한 React 코드 스타일 유지

<br>

**2) 백엔드 데이터 접근의 단순화: Full Access to the Backend**

- 문제
  - React에서 데이터 접근은 복잡함.
    - UI를 위해 별도 API를 만들거나, 기존 API를 UI 목적에 맞게 재사용해야 하는 경우가 많음.
  - 특히 앱이 커질수록 데이터 접근 구조가 복잡해짐.
- 해결
  - Server Components는 서버에서 파일 시스템, DB, 마이크로서비스 등 백엔드 데이터 소스에 직접 접근할 수 있음.
  - UI와 데이터 접근 로직을 한 곳에서 관리할 수 있음.
- 예시

  ```JSX
  // 파일 시스템에서 데이터 읽기
  import fs from 'fs';

  export default async function Note({ id }) {
    const note = JSON.parse(await fs.readFile(`${id}.json`));
    return <NoteWithMarkdown text={note.text} />;
  }

  // DB에서 데이터 읽기
  import db from 'db';

  export default async function Note({ id }) {
    const note = await db.notes.get(id);
    return <NoteWithNoteWithMarkdown note={note} />;
  }
  ```

- 효과
  - 데이터 접근 로직 단순화
  - 앱 규모가 커져도 구조 유지 용이
  - 신규 개발자도 쉽게 이해 가능

<br>

**3) 자동 코드 분할: Automatic Code Splitting**

- 문제
  - 기존 코드 분할은 개발자가 수동으로 React.lazy 등을 사용해야 했고, 렌더링이 지연되어 성능 이점이 줄어들었음.
- 해결
  - Server Components는 자동으로 코드 분할을 수행함.
  - Server Component는 클라이언트 컴포넌트를 스트리밍할 때, 필요한 부분만 자동으로 코드 분할하여 전송함.
- 예시

  ```JSX
  // Server Component
  import OldPhotoRenderer from './OldPhotoRenderer';
  import NewPhotoRenderer from './NewPhotoRenderer';

  export default function Photo(props) {
    if (FeatureFlags.useNewPhotoRenderer) {
      return <NewPhotoRenderer {...props} />;
    } else {
      return <OldPhotoRenderer {...props} />;
    }
  }
  ```

- 효과
  - 코드 분할 자동화
  - 렌더링 시작 시점을 앞당겨 지연 최소화
  - 개발자는 자연스럽게 코드 작성 가능

**4) 성능 저하 요인 제거: No Client-Server Waterfalls**

- `Client-Server Waterfalls 현상이란?`
  - 요청 → 응답 → 그 다음 요청 → 응답 → 그 다음 요청 ...
  - 위와 같이 순차적으로 요청이 이어지는 현상. 즉, 하나가 끝나야 다음이 시작되는 구조를 말하며, 전체 로딩 시간이 계단식으로 길어지는 문제.
- 문제
  - 기존에는 클라이언트에서 순차적으로 데이터를 요청해 워터폴 현상이 발생함.
  - 부모-자식 컴포넌트가 모두 useEffect로 데이터를 가져오면, 부모가 끝나야 자식이 데이터를 fetch하고 렌더링이 시작됨.
- 해결
  - Server Components는 서버에서 데이터 조회를 수행하므로, 클라이언트 워터폴을 제거함.
    - 브라우저가 아니라 서버에서 실행됨.
    - useEffect와 브라우저 JS가 필요없고, fetch를 서버에서 바로 실행가능
    - CSR vs RSC
      - CSR
        - HTML → JS 다운로드 → JS 실행 → fetch
          ```JSX
            useEffect(() => {
              fetch("/api/user").then(...).then(...)
            }, [])
          ```
        - 클라이언트와 서버가 요청과 응답을 주고 받으며 딜레이가 발생함.
      - RSC
        - 서버에서 fetch → 완성된 결과 전송
          ```JSX
            export default async function Page() {
              const user = await fetch("/api/user").then(...).then(...)
            }
          ```
        - 서버에서 fetch하여 데이터가 포함된 HTML을 바로 전송함. 브라우저는 데이터를 기다릴 필요없음
  - 서버에서 필요한 데이터를 미리 가져와 렌더링함.
- 예시
  ```JSX
  // Server Component
  export default async function Note(props) {
    const note = await db.notes.get(props.id);
    return <NoteWithMarkdown note={note} />;
  }
  ```
- 효과
  - 렌더링 지연 감소
  - 데이터 접근 효율 향상
  - 사용자 경험 개선

<br>

**5) 추상화 비용 감소: Avoiding the Abstraction Tax**

- 문제
  - 복잡한 추상화 계층은 코드와 런타임 오버헤드를 증가시킴.
  - JavaScript는 AOT(Ahead-of-Time) 컴파일이 어려워, 추상화 비용을 줄이기 어려움.
- 해결
  - Server Components는 서버에서 렌더링 후 최종 DOM만 클라이언트에 전송함. 즉, 중간 추상화 계층은 서버에서 사라지고, 클라이언트는 가볍게 렌더링만 수행함.
- 예시

  ```JSX
    // Server Component
    export default async function Note({ id }) {
    const note = await db.notes.get(id);
    return <NoteWithMarkdown note={note} />;
    }

    // NoteWithMarkdown.js
    export default function NoteWithMarkdown({ note }) {
    const html = sanitizeHtml(marked(note.text));
    return <div dangerouslySetInnerHTML={{ __html: html }} />;
    }

    // 클라이언트는 최종적으로 <div>만 받음
  ```

- 효과
  - 추상화 비용 제거
  - 번들 크기 감소
  - 렌더링 효율 향상

<br>

**6) 서버 렌더링과 클라이언트 렌더링의 통합: Unified Solution**

- 문제
  - 순수 서버 렌더링은 정적 콘텐츠에 유리하지만, 인터랙션에는 한계가 있음.
  - 순수 클라이언트 렌더링은 인터랙션에 강하지만, 초기 로딩과 데이터 접근이 비효율적임.
  - 두 방식을 혼합하면 언어, 프레임워크, 생태계가 분리되어 복잡해짐.
- 해결
  - Server Components는 하나의 언어(JS/TS)와 프레임워크(React) 안에서 서버와 클라이언트의 장점을 결함함.
  - 서버는 정적 콘텐츠를 빠르게 렌더링하고, 클라이언트는 인터랙션과 상태 관리를 담당함.
- 효과
  - 개발 생산성 향상
  - 유지보수성 향상
  - 성능과 사용자 경험 개선
