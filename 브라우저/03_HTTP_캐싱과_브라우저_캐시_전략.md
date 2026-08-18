### 03. HTTP 캐싱과 브라우저 캐시 전략

---

> HTTP 캐싱은 한 번 받아온 리소스를 재사용해서, 같은 요청을 서버까지 왕복하지 않고도 처리하는 메커니즘이다.  
> 언제까지 캐시를 그대로 믿을지(freshness), 캐시가 여전히 유효한지 서버에 물어볼지(validation)를 `Cache-Control`, `ETag`, `Last-Modified` 같은 응답 헤더로 조율한다.

<br>

#### 1. 두 갈래의 캐시 정책 — Freshness와 Validation

---

- 01번 노트의 "요청과 응답" 단계에서 브라우저는 HTML을 파싱하며 CSS, JS, 이미지를 추가로 요청한다고 했다. 캐싱이 없다면 새로고침할 때마다, 심지어 페이지를 이동할 때마다 이 리소스들을 매번 새로 받아와야 한다.
- HTTP는 이 문제를 두 가지 다른 전략으로 해결한다.
  | 전략 | 동작 | 대표 헤더 |
  |---|---|---|
  | **Freshness (신선도)** | 유효기간 안에는 서버에 묻지도 않고 캐시를 그대로 쓴다 | `Cache-Control: max-age` |
  | **Validation (검증)** | 유효기간이 지나면(stale) 서버에 "여전히 유효하냐"만 확인하고, 맞으면 본문 없이 재사용한다 | `ETag`, `Last-Modified` + 조건부 요청 |
- 즉 캐시된 리소스는 신선(fresh) → 오래됨(stale) → (검증 성공 시) 다시 신선, 순서로 상태가 바뀐다. **stale이 됐다고 곧바로 버려지는 게 아니라, 검증을 거쳐 재사용될 기회를 한 번 더 얻는다**는 점이 핵심이다.

<br>

#### 2. `Cache-Control` 지시어 뜯어보기

---

- `Cache-Control`은 서버가 응답에 실어 보내는, 캐시 정책을 정의하는 핵심 헤더다. 실무에서 자주 쓰는 지시어는 다음과 같다.
  | 지시어 | 의미 |
  |---|---|
  | `max-age=<초>` | 이 시간(초) 동안은 fresh로 간주하고 재검증 없이 그대로 사용 |
  | `no-cache` | 캐시에 저장은 하되, **매번 서버에 검증(재확인)을 거친 뒤 사용** — 이름과 달리 "캐시 안 함"이 아니다 |
  | `no-store` | 아예 캐시하지 않음. 요청·응답 자체를 저장 금지 (민감 정보 응답에 사용) |
  | `private` / `public` | private는 브라우저 개인 캐시만 허용, public은 CDN 같은 중간 캐시 서버도 저장 허용 |
  | `immutable` | 유효기간 내에는 리소스가 절대 바뀌지 않는다고 선언 — 새로고침 시 재검증 요청조차 생략 |
  | `stale-while-revalidate=<초>` | stale이 된 뒤에도 지정 시간 동안은 일단 캐시를 즉시 반환하고, 백그라운드에서 조용히 최신 버전을 받아와 다음 요청부터 반영 |
- 가장 많이 헷갈리는 두 지시어는 `no-cache`와 `no-store`다. **`no-cache`는 "캐시하지 마라"가 아니라 "캐시는 하되 쓰기 전에 항상 서버에 확인해라"** 는 뜻이고, 완전히 캐시를 금지하려면 `no-store`를 써야 한다.

<br>

#### 3. 검증 캐싱 — `ETag`/`Last-Modified`와 304 Not Modified

---

- `max-age`가 지나 stale 상태가 된 리소스도, 서버의 실제 내용이 그대로라면 굳이 본문 전체를 다시 받을 필요가 없다. 이때 브라우저는 **조건부 요청(conditional request)** 을 보내 "내가 가진 버전이 아직 최신이면 본문 없이 그렇다고만 알려달라"고 서버에 묻는다.
- 서버는 응답에 리소스 버전을 식별할 수 있는 정보를 함께 실어 보내고, 브라우저는 다음 요청 때 그 값을 그대로 되돌려준다.

  | 응답 헤더 (최초 응답) | 요청 헤더 (재검증) | 의미 |
  |---|---|---|
  | `ETag: "abc123"` | `If-None-Match: "abc123"` | 리소스의 지문(해시 등)이 같은지 비교 |
  | `Last-Modified: Wed, 01 Jul 2026 ...` | `If-Modified-Since: Wed, 01 Jul 2026 ...` | 마지막 수정 시각이 그대로인지 비교 |

- 서버가 비교해서 리소스가 안 바뀌었다고 판단하면 본문 없이 `304 Not Modified`만 응답한다. 바뀌었다면 평소처럼 `200 OK`와 함께 새 본문을 내려준다.

```
# 브라우저 → 서버 (재검증 요청)
GET /style.css HTTP/1.1
If-None-Match: "abc123"

# 서버 → 브라우저 (변경 없음)
HTTP/1.1 304 Not Modified
ETag: "abc123"
# 본문 없이 헤더만 — 브라우저는 캐시에 있던 style.css를 그대로 재사용한다
```

- `ETag`와 `Last-Modified`는 함께 쓰일 수 있는데, 왜 굳이 둘 다 필요할까? `Last-Modified`는 초 단위까지만 표현되기 때문에 1초 안에 여러 번 바뀌어도 감지하지 못하고, 분산 서버 환경에서는 서버마다 파일 수정 시각이 미묘하게 어긋날 수 있다. `ETag`는 내용 자체(또는 내용의 해시)를 기준으로 비교하므로 더 정확하지만, 계산 비용이 조금 더 든다. 그래서 RFC 9110도 가능하면 둘 다 함께 보내도록 권장한다.

<br>

#### 4. 실무 전략 — 정적 자산은 `immutable`, HTML은 `no-cache`

---

- 02번 노트에서 다룬 Webpack 번들링과 캐싱 전략은 실무에서 짝을 이룬다. 빌드 산출물 파일명에 콘텐츠 해시를 넣는 이유가 바로 캐시 전략과 직결되기 때문이다.

```
dist/
 ├─ index.html
 ├─ main.a1b2c3d4.js
 └─ style.e5f6g7h8.css
```

- 파일 내용이 바뀌면 해시값도 함께 바뀌어 **URL 자체가 달라지므로**, "이 파일을 오래 캐시해도 되는가"라는 고민 없이 영구히 캐시해도 안전하다. 반대로 절대 바뀌지 않아야 할 `index.html`은 항상 최신 상태를 보장해야 한다.

  | 리소스 | 권장 정책 | 이유 |
  |---|---|---|
  | `index.html` | `Cache-Control: no-cache` | 파일명이 고정이라 새 배포를 감지하려면 매번 서버 검증이 필요하다 |
  | 해시가 포함된 JS/CSS 번들 | `Cache-Control: max-age=31536000, immutable` | URL이 내용과 1:1로 묶여 있어 영원히 캐시해도 안전하다 |
  | 로고, 파비콘처럼 자주 안 바뀌는 이미지 | `Cache-Control: max-age=<적당한 기간>` | 가끔 바뀔 수 있으니 무한 캐시보다는 유효기간을 둔다 |

- 이 조합 덕분에 사용자는 재방문 시 `index.html`만 가볍게 재검증하고, 실제로 무거운 JS/CSS 번들은 내용이 안 바뀌었다면 네트워크 요청 자체가 발생하지 않는다.

<br>

#### 5. Service Worker 캐시 — HTTP 캐시와는 다른 레이어

---

- 지금까지의 캐싱은 브라우저가 HTTP 스펙에 따라 **자동으로** 관리하는 캐시다. 반면 Service Worker의 `Cache Storage API`는 **개발자가 코드로 직접** 무엇을 언제 캐시할지 제어하는, 완전히 별도의 저장소다. 오프라인 지원이 필요한 PWA에서 특히 중요하다.
- 대표적인 캐싱 전략 3가지는 다음과 같다.
  | 전략 | 동작 | 적합한 대상 |
  |---|---|---|
  | **Cache First** | 캐시에 있으면 그대로 반환, 없을 때만 네트워크 요청 | 거의 안 바뀌는 정적 자산 (이미지, 폰트) |
  | **Network First** | 네트워크를 우선 시도하고, 실패(오프라인 등)하면 캐시로 폴백 | 최신성이 중요한 API 응답, HTML |
  | **Stale-While-Revalidate** | 캐시를 즉시 반환하면서, 동시에 백그라운드에서 네트워크로 최신 버전을 받아 다음 요청을 위해 캐시를 갱신 | 속도와 최신성을 적당히 절충하고 싶은 리소스 |

```javascript
// Service Worker 내부 — Cache First 전략 예시
self.addEventListener("fetch", (event) => {
  event.respondWith(
    caches.match(event.request).then((cached) => {
      // 캐시에 있으면 그대로 반환, 없으면 네트워크 요청 후 캐시에 저장
      return cached || fetch(event.request).then((response) => {
        const clone = response.clone();
        caches.open("v1").then((cache) => cache.put(event.request, clone));
        return response;
      });
    }),
  );
});
```

- 같은 이름의 `Cache-Control` 헤더가 있어도, Service Worker가 `fetch` 이벤트를 가로채 응답을 직접 반환하면 그 요청은 HTTP 캐시 로직을 거치지 않고 Service Worker 캐시가 우선한다. 즉 두 캐시 레이어는 서로 독립적으로 동작하며, 개발자가 명시적으로 등록한 Service Worker가 있다면 그쪽이 더 앞단에서 요청을 가로챈다.

<br>

#### 6. 정리

---

- HTTP 캐싱은 **신선도(freshness)** 로 재검증 자체를 생략하거나, **검증(validation)** 으로 본문 전송만 생략하는 두 축으로 동작한다.
- `no-cache`는 "캐시는 하되 매번 검증", `no-store`는 "아예 캐시 금지"로 이름과 달리 헷갈리기 쉬우니 구분해서 써야 한다.
- `ETag`/`Last-Modified` + 조건부 요청은 리소스가 안 바뀌었을 때 `304 Not Modified`로 본문 전송을 생략해 대역폭을 아낀다.
- 실무에서는 **콘텐츠 해시가 담긴 정적 자산은 `immutable`로 영구 캐시**, **엔트리 HTML은 `no-cache`로 항상 재검증**하는 조합이 표준적인 배포 전략이다.
- Service Worker의 Cache Storage API는 HTTP 캐시와 별개의 레이어이며, Cache First / Network First / Stale-While-Revalidate 전략을 코드로 직접 제어해 오프라인 지원까지 확장할 수 있다.
