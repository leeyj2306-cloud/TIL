# 에어비앤비 웹사이트 분석 (REA 런타임 캡처)

- **분석일:** 2026-11-02
- **대상:** `https://www.airbnb.com/`, `https://www.airbnb.com/s/Seoul/homes`
- **도구:** REA `capture_browser_scenario`

## 요약

에어비앤비 검색 페이지는 **서버가 검색 결과를 JSON으로 HTML에 넣어 보내고, 브라우저의 React가 이를 이어받아 화면을 그리는 구조**로 보입니다. 첫 화면을 그리기 위한 별도의 검색 API 호출은 관찰되지 않았습니다.

## 분석 범위와 방법

- 로그인 없이 임시 프로필의 헤드리스 Chrome으로 공개 페이지 두 개만 열었습니다.
- 클릭이나 입력은 하지 않았고, 페이지를 연 뒤 약 6초씩 대기했습니다.
- 언어는 en, 통화는 KRW입니다.
- 봇 차단 없이 실제 숙소 목록과 지도가 표시됐습니다.

| 캡처 | 대상 | 수집 내용 | 상태 |
|---|---|---|---|
| 1차 | 홈 → 서울 검색 | 네트워크, 워커, 프레임, 스크린샷 | 이벤트 누락 없음. 미완료 요청 7건 때문에 `incomplete` |
| 2차 | 서울 검색 | 헤더 값, 응답 본문, 저장소 | 예산 한도로 일부만 보존. 요청 130건 보존, 197건 누락 |

## 관찰한 사실

### 렌더링과 데이터

| 항목 | 관찰 내용 |
|---|---|
| HTML 크기 | 914KB |
| 초기 상태 JSON | `data-deferred-state-0`, 377KB |
| 검색 데이터 | HTML 안에 `StaysSearch` 결과 18건과 지도 핀 20건 포함 |
| 숙소 링크 | 초기 HTML에 `/rooms/` 링크 없음 |
| 페이지네이션 | Base64 커서 사용. 디코딩 예: `{"section_offset":0,"items_offset":18,"version":1}` |
| 페이지당 항목 | 18건 (`itemsPerGrid`) |
| 실험 플래그 | 검색 요청에 `treatmentFlags` 12개 |

검색 라우트는 `/s/:location?/homes/:additionalRefinements?`로, `/s/:location?/hotels/...`와 같은 항목에 묶여 있습니다.

### API

- **GraphQL:** `/api/v3/{작업명}/{sha256}` 형태의 Persisted Query이며, `extensions`에 `sha256Hash`를 담아 보냅니다.
- **REST:** `/api/v2/*` 경로가 함께 쓰입니다.

| 종류 | 관찰된 작업 |
|---|---|
| GraphQL | `AutoSuggestionsQuery`, `GetConsentFlagsQuery`, `GetConsentForUserQuery`, `Header`, `LocationContextPlacesQuery`, `NaviServerAnnouncementsQuery`, `PromotionsByPlacementsQuery` |
| REST | `client_configs`, `get-data-layer-variables`, `marketing_event_tracking`, `user_markets` |

### 프런트엔드

| 항목 | 관찰 내용 |
|---|---|
| 번들 | `metroRequire`, `asyncRequire` 존재. `__d(` 모듈 등록 5,277회 |
| React | 스크립트에 `18.3.1` 문자열 존재 |
| CSS | `aphrodite-classes`와 `data-linaria-css` 공존 |
| 스크립트 규모 | 두 페이지에서 요청 227건. 보존된 94개 파일만 약 7.6MB(디코딩 기준) |
| 정적 자산 | `a0.muscache.com` |
| 지도 | Google Maps 벡터 타일과 WASM 라벨러. CSP에는 Mapbox도 남아 있음 |

보존된 파일 중 큰 스크립트는 다음과 같습니다.

| 파일 | 크기(바이트) |
|---|---:|
| `asyncRequire.8caee46204.js` | 565,877 |
| `client.8073d3e024.js` | 436,868 |
| `92bd.0eaee16ec0.js` | 338,229 |
| `2147.0084f6e96c.js` | 278,455 |
| `StaysSearchRoute.10f089128d.js` | 244,537 |

### 네트워크 규모 (1차 캡처)

| 페이지 | 요청 수 | 스크립트 | XHR | fetch | 이미지 |
|---|---:|---:|---:|---:|---:|
| 홈 | 211 | 115 | 15 | 29 | 30 |
| 서울 검색 | 185 | 112 | 23 | 13 | 19 |

### 인프라와 보안

- **응답 헤더:** `server: nginx`, `x-envoy-upstream-service-time`, `akamai-request-bc`, `x-kraken-loop-name: core-guest-loop`
- **응답 시간:** CDN MISS, edge 47ms, origin 204ms
- **HTML 캐시:** `no-store, max-age=0, private, must-revalidate`
- **봇 탐지:** 자사 서브도메인 `d0a7e.airbnb.com/tags.js`에서 DataDome v5.2.0을 불러오고, `datadome` 쿠키 설정
- **HSTS:** `max-age=63072000; includeSubDomains; preload`
- **기타 헤더:** `X-Frame-Options: SAMEORIGIN`, `X-Content-Type-Options: nosniff`, HTTP/3 광고(`alt-svc: h3`)
- **CSP:** `script-src`에 소스 83개, `'unsafe-eval'` 허용, `/tracking/csp`로 위반 보고

### 추적

- **자사 수집:** `/tracking/jitney/logging/messages` 18회, `/tracking/airdog` 6회, `/sgtm/g/collect` 3회
- **제3자 요청:** Google Ads/Analytics, The Trade Desk, Facebook, Snapchat, Pinterest, Naver, Rubicon, Casale Media
- **동의 없이 발생:** 아무 조작을 하지 않았는데도 이 요청들이 발생했고, 일부 요청에는 `gdpr=0`이 실려 있음
- **쿠키:** 32개 설정, 그중 `HttpOnly`는 `FPID`와 `_gtmeec` 2개뿐

## 추론한 내용

아래 항목은 직접 확인한 사실이 아니라 관찰 결과로 미루어 판단한 것입니다.

- **렌더링:** 숙소 카드는 브라우저에서 JSON으로 그리는 것으로 보입니다. `/rooms/` 링크가 초기 HTML에 없다는 점이 근거입니다.
- **사내 프레임워크:** 이름과 위치만 보고 다음과 같이 판단했습니다.
  - Hyperloop: SSR/라우팅
  - Niobe: GraphQL 클라이언트
  - Jitney: 이벤트 로깅
  - Kraken: 서버 계층
- **요청 경로:** Akamai → nginx/Envoy → Node SSR 순서로 보이지만, 헤더만으로 순서를 확정할 수는 없습니다.
- **스타일 전환:** Aphrodite에서 Linaria로 옮겨 가는 중일 수 있습니다.
- **`/sgtm`:** 자사 도메인을 거치는 Google 태그 경로로 보입니다.

## 확인하지 못한 부분

- **미탐색 영역:** 로그인, 숙소 상세, 예약·결제, 호스트 화면, 필터 변경이나 다음 페이지 요청은 보지 않았습니다.
- **WebSocket:** CSP에 `wss://ws.airbnb.com`이 있지만 이번 세션에서 연결은 관찰되지 않았습니다.
- **스크립트 분석:** 2차 캡처 일부만 분석한 부분 표본이라, 문자열 출현 횟수는 전체 값이 아닙니다.
- **쿠키와 저장소:** 값은 해시로만 보존되어 내용을 확인하지 않았고, 이름과 속성만 확인했습니다.
- **대표성:** 한 번의 헤드리스 실행이라 지역이나 A/B 실험에 따라 결과가 달라질 수 있습니다.

## 증거 자료

| 내용 | Evidence ID |
|---|---|
| 홈+검색 네트워크 | `ev_a3836a228a4a57809718e90e7535086eb6256a8c6cdea20e996f4f2ee640608a` |
| 검색 헤더+본문 | `ev_d2bc98b71712b88596d981ab7912001f661f6c53e63cb66ebe299e739f78c803` |

원본 파일은 `airbnb-site-analysis` 폴더에 있습니다.

- [search.png](airbnb-site-analysis/search.png): 검색 페이지 스크린샷
- `evidence.json`: REA 증거 번들 (로컬 보관, 저장소 제외)
- `search.html`: 검색 페이지 HTML (로컬 보관, 저장소 제외)
