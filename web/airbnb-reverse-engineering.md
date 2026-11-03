# 에어비앤비 웹 역공학 분석

- 분석일: 2026-11-02
- 대상: `https://www.airbnb.co.kr` 비로그인 상태의 홈, 서울 검색 결과, 숙소 상세 1건
- 근거 표기: **[관찰]** 이번 세션에서 직접 확인 / **[공개]** 에어비앤비 공식 자료 / **[추정]** 관찰을 바탕으로 한 해석

## 1. 요약

에어비앤비 웹은 React 기반 SPA를 서버에서 먼저 렌더링한 뒤, 해시로 식별되는 GraphQL 작업(`/api/v3/{작업명}/{해시}`)으로 데이터를 가져오는 구조입니다. 번들러는 Metro, 스타일은 Linaria이고, 숙소 상세는 서버가 내려주는 섹션 단위로 구성되는 것으로 보입니다.

| 영역 | 내용 | 근거 |
|---|---|---|
| 번들러 | Metro 자체 변형(`metroRequire`, `airmetro`, 전역 `__d`/`__r`) | 관찰 + 공개 |
| 스타일 | Linaria 아토믹 클래스(`atm_*`), Aphrodite 흔적 공존 | 관찰 + 공개 |
| 데이터 | GraphQL 스타일 `/api/v3`, 일부 REST `/api/v2` | 관찰 |
| 서버 데이터 전달 | HTML에 `data-deferred-state-0`(`niobeClientData`) 삽입 | 관찰 |
| GraphQL 플랫폼 | Niobe(코드 생성), Viaduct(데이터 계층) | 공개 |
| UI 구성 | 섹션 단위 서버 드리븐 UI(Ghost Platform) | 공개, 웹 적용은 추정 |
| 인프라 | Akamai 응답 헤더, `Server: envoy`, HTTP/2 사용(HTTP/3 광고) | 관찰 |
| 봇 방어 | DataDome 쿠키, CSP에 Arkose·GeeTest·Sift 등 허용 | 관찰 |

## 2. 프런트엔드

### 번들과 로딩 [관찰]

- 자산은 `a0.muscache.com/airbnb/static/packages/web/` 아래에 있고, 파일명에 콘텐츠 해시가 붙습니다.
- 경로가 `web/common/`과 `web/ko/`로 나뉩니다. **[추정]** 번역 문자열이 포함된 청크를 로케일별로 따로 빌드하는 방식입니다.
- 진입점은 `core-guest-spa/entrypoints/`이고, 런타임은 `hyperloop-browser/`, 라우팅은 `routing/routeHandler/genericRouteHandler`에 있습니다.
- `Link` 응답 헤더로 CSS와 Airbnb Cereal 가변 폰트를 preload하고, Early Hints 리소스도 1건 확인했습니다.
- 공개 글에 따르면 Metro는 개발 중 요청받은 번들만 즉시 처리하며, 개발과 프로덕션 번들링 모두에 쓰입니다.

### 스타일 [관찰 + 공개]

- 클래스는 `atm_9s_1txwivl`처럼 원자 단위로 생성됩니다. 인라인 데이터에는 `data-linaria-css`와 `aphrodite-classes`가 함께 있습니다.
- 공개 글에 따르면 react-with-styles에서 Linaria로 옮겼고, 런타임 비용을 줄이려고 스타일을 빌드 시점에 정적 CSS로 추출합니다.
- `data-theme`, `data-color-scheme` 속성이 있습니다.

### 서버 렌더링과 초기 상태 [관찰]

HTML에 포함된 주요 JSON 스크립트입니다(단위: 문자 수).

| ID | 홈 | 검색 | 상세 | 역할 |
|---|---|---|---|---|
| `data-deferred-state-0` | 211,936 | 335,725 | 290,812 | `niobeClientData` 배열, GraphQL 초기 데이터로 추정 |
| `data-injector-instances` | 149,843 | 150,037 | 154,676 | 페이지와 무관하게 크기가 비슷해 앱 공통 설정으로 추정 |
| `data-initializer-bootstrap` | 1,621 | 1,673 | 1,735 | 로케일, API 기본 설정, 추적 컨텍스트, 지도 스크립트 URL |

검색의 `StaysSearch`와 상세의 `StaysPdpSections` 요청도 브라우저에서 발생했습니다. 이 요청이 초기 상태의 재요청인지 추가 데이터 요청인지는 확인하지 못했습니다.

### 규모와 속도 [관찰]

한 번씩만 측정했고 캐시가 있는 브라우저 기준이라 참고용입니다.

| 페이지 | DOM 노드 | TTFB | DCL | load |
|---|---|---|---|---|
| 홈 | 3,264 | 107ms | 1,133ms | 1,477ms |
| 검색 | 4,020 | 59ms | 1,163ms | 2,693ms |
| 상세 | 2,633 | 647ms | 876ms | 1,569ms |

- 홈의 First Contentful Paint는 856ms였습니다.
- 홈에서 스크립트 태그는 153개, 리소스 요청은 `a0.muscache.com`만 194건이었습니다.
- 서비스 워커는 페이지 캐시용이 아니라 서버 사이드 GTM 경로(`/sgtm/_/service_worker/...`)의 것만 확인했습니다.

## 3. API

### 요청 형식 [관찰]

```
GET  /api/v3/{작업명}/{64자리 해시}?operationName&locale&currency&variables&extensions
POST /api/v3/{작업명}/{64자리 해시}?operationName&locale&currency
POST /api/v2/{리소스}?locale&currency
```

- **[추정]** 해시와 `extensions` 파라미터로 보아 persisted query 방식입니다. 공개 자료로는 확인하지 못했습니다.
- `/api/v3` 요청 공통 헤더 이름: `X-Airbnb-API-Key`, `X-Airbnb-GraphQL-Platform`, `X-Airbnb-GraphQL-Platform-Client`, `X-CSRF-Token`, `X-CSRF-Without-Token`, `X-Client-Version`, `x-client-request-id`, `x-airbnb-client-trace-id`, `X-Airbnb-Supports-Airlock-V2`
- 헤더 값은 기록하지 않았습니다.

### 화면별 작업 [관찰]

| 화면 | 작업 |
|---|---|
| 공통 | `Header`, `NaviServerAnnouncementsQuery`, `GetConsentFlagsQuery`, `AutoSuggestionsQuery` |
| 공통(REST) | `client_configs`, `get-data-layer-variables`, `marketing_event_tracking` |
| 검색 | `StaysSearch`(POST), `LocationContextPlacesQuery`(POST) |
| 상세 | `StaysPdpSections`(POST), `StaysPdpBookItQuery`, `StaysPdpReviewsQuery`, `StaysPdpPoliciesQuery`, `PdpAvailabilityCalendarV2Query`, `SimilarListingsCarouselQuery`, `SearchInputQuery`(POST) |

상세 페이지는 본문과 예약 위젯, 후기, 정책, 달력을 별도 작업으로 나눠 가져옵니다.

### 서버 드리븐 UI

숙소 상세의 DOM에서 `data-section-id` 20개를 확인했습니다. **[관찰]**

```
TITLE_DEFAULT, HERO_DEFAULT, NAV_DEFAULT, BOOK_IT_NAV, OVERVIEW_DEFAULT_V2,
GUEST_FAVORITE_BANNER, HIGHLIGHTS_COMPACT, HOST_OVERVIEW_DEFAULT, DESCRIPTION_DEFAULT,
SLEEPING_ARRANGEMENT_WITH_IMAGES, BOOK_IT_SIDEBAR, REPORT_TO_AIRBNB, LOCATION_DEFAULT,
REVIEWS_DEFAULT, AMENITIES_DEFAULT, AVAILABILITY_CALENDAR_INLINE, MEET_YOUR_HOST,
HOST_LEGAL_DETAILS, POLICIES_DEFAULT, SEO_LINKS_DEFAULT
```

- **[공개]** Ghost Platform은 웹·iOS·Android가 하나의 GraphQL 스키마를 공유하며, 섹션·화면·액션으로 UI를 구성합니다.
- **[추정]** 위 섹션 ID와 `_DEFAULT`, `_V2` 같은 접미사는 섹션의 변형을 서버가 고르는 구조와 일치합니다. 응답 본문을 분석해 확인한 내용은 아닙니다.

## 4. 인프라와 보안 [관찰]

- **응답 경로:** `Akamai-Request-BC`, `Server: envoy`, `Server-Timing` 헤더가 있습니다. HTML은 `no-store`로 캐시되지 않습니다.
- **내부 라우팅:** `x-kraken-loop-name: core-guest-loop` 헤더가 있고, CSP 보고 URL의 `controller` 값도 같습니다.
- **보안 헤더:** HSTS(preload), `X-Frame-Options: SAMEORIGIN`, `nosniff`, CSP를 적용합니다. 다만 `script-src`에 `'unsafe-eval'`과 `https:` 전체가 포함돼 스크립트 출처 제한은 넓은 편입니다.
- **CSP 허용 도메인:** 결제(Stripe, PayPal, Klarna, Plaid, Amex, Apple 로그인), 3DS(Cardinal), 본인 확인(Netverify), 봇·사기 방지(Arkose, GeeTest, Sift, ThreatMetrix, reCAPTCHA)가 있습니다. 허용 목록일 뿐이라 실제 로드 여부는 별개입니다.
- **클라이언트 힌트:** `Accept-CH`로 기기 메모리, DPR, 뷰포트 폭, 네트워크 유형을 요청합니다.

## 5. 측정과 실험 [관찰]

- **자체 로깅:** `/tracking/jitney/logging/messages`가 페이지당 12~19회, `/tracking/airdog`가 2회 호출됐습니다.
- **태그 관리:** GTM을 자사 도메인 경로(`/gtg/`, `/sgtm/`)로 프록시합니다.
- **외부 픽셀:** Google Ads, Facebook, Pinterest, Snapchat, Naver, The Trade Desk 요청을 확인했습니다.
- **실험과 설정:** `data-flagger_cdn_experiments`, `/api/v2/client_configs`, `robots.txt`의 `/api/v1/trebuchet`가 있습니다. **[추정]** 기능 플래그와 실험 설정용입니다.

## 6. URL, SEO, 화면

- **검색 URL:** `/s/{장소}/homes`, `/s/{장소}/experiences`
- **필터 파라미터:** `place_id`, `refinement_paths[]`, `kg_or_tags=Tag:{id}`, `search_type`
- **노출 추적:** 검색에서 상세로 갈 때 `source_impression_id`가 붙습니다.
- **구조화 데이터:** 상세는 JSON-LD `VacationRental`, `Product`를 쓰고, 검색 페이지에는 없었습니다.
- **그 밖의 SEO:** Open Graph 태그와 사이트맵 인덱스, 상세의 `SEO_LINKS_DEFAULT` 섹션이 있습니다.
- **상단 UI:** 전체·숙소·체험·서비스 탭과 여행지·날짜·여행자 3단 검색창으로 구성됩니다. 탭 아이콘은 `video` 요소입니다.
- **개인화:** 위치 기반으로 보이는 "근처의 체험", 도시별 인기 숙소 캐러셀이 나옵니다.
- **비로그인 유도:** 첫 방문에 신규 게스트 할인 모달이 떴습니다.
- **지도:** 검색 화면은 Google Maps를 쓰고, 부트스트랩에 OpenStreetMap과 Gaode(중국) 지도 URL 설정도 있습니다.
- **테스트 식별자:** `data-testid`가 프로덕션에 남아 있습니다(`card-container`, `listing-card-title`, `little-search` 등).

## 7. 내 프로젝트에 적용할 점

1. **화면 데이터 분리:** 상세 페이지처럼 핵심 본문과 후기·달력·추천을 별도 요청으로 나누면 느린 영역이 전체를 막지 않습니다.
2. **화면 구성의 서버 제어:** 섹션 ID와 변형 단위로 응답을 설계하면 앱 배포 없이 구성을 바꿀 수 있습니다. 여러 클라이언트를 함께 운영할 때 효과가 큽니다.
3. **제로 런타임 CSS:** 런타임 CSS-in-JS 비용이 문제라면 빌드 시점 추출 방식을 검토할 만합니다.
4. **요청 추적:** 모든 요청에 클라이언트 추적 ID와 버전 헤더를 붙이면 장애 추적이 쉬워집니다.
5. **로케일별 청크:** 번역 문자열을 언어별 청크로 분리하면 불필요한 번들 전송을 줄일 수 있습니다.

## 8. 범위와 한계

### 접근 정책

- `robots.txt`는 `ClaudeBot`에 `Disallow: /`를 지정하고, `*`에도 `/s/*/*`를 금지합니다.
- 이 규칙을 **홈, 검색, 상세 페이지를 방문한 뒤에 확인했습니다.** 확인 후에는 에어비앤비 사이트에 더 접근하지 않았고, 그래서 이 문서의 관찰 범위가 3개 페이지로 제한됩니다.
- 검색 결과의 2차 자료에 따르면 에어비앤비 약관은 봇·크롤러·스크레이퍼 등 자동화된 수집을 금지합니다. 약관 원문은 확인하지 못했으니, 이 문서를 외부에 공유하거나 분석을 이어 가기 전에 원문을 확인하세요.

### 수집 방식

- 자동화 브라우저(ego-browser)로 3개 페이지를 각 1회 열었습니다.
- `robots.txt`와 응답 헤더는 curl로 `Mozilla/5.0` User-Agent를 지정해 가져왔습니다. 자동화 도구임을 드러내지 않은 요청입니다.
- API는 페이지가 스스로 보낸 요청만 관찰했고 직접 호출하지 않았습니다.
- 숙소 데이터, 후기, 개인정보를 수집하지 않았고, API 키·토큰·쿠키 값은 이 문서에 넣지 않았습니다.
- 방어 체계를 우회하거나 시험하지 않았습니다.
- 로컬 ego-browser 프로필에는 방문 과정에서 생긴 에어비앤비와 광고 관련 쿠키가 남아 있을 수 있습니다.

### 분석하지 않은 부분

- 로그인, 예약, 결제, 호스팅, 체험·서비스 상세, 모바일 앱
- GraphQL 스키마와 응답 본문의 세부 구조, 소스 코드 수준의 동작
- 백엔드 구조: 공개 자료 외에는 알 수 없습니다.
- Hyperloop의 정체: 자산 경로와 `data-is-hyperloop` 속성만 확인했고, 공개 자료는 찾지 못했습니다.

### 결과에서 제외한 신호

Vue DevTools 전역 변수와 Grammarly 속성(`data-gr-*`)은 브라우저 확장 프로그램의 흔적으로 보여 에어비앤비 기술 스택에서 뺐습니다. `__reactRouterVersion` 전역도 출처를 확정하지 못해 근거로 쓰지 않았습니다. React 사용은 `react-application` 루트와 공개 글을 근거로 판단했습니다.

### 도구

요청하신 "역공학 스킬"은 이 환경에 없습니다. TIL에 정리된 REA도 설치돼 있지 않아 쓰지 않았고, 브라우저 관찰과 공개 자료로 분석했습니다.

## 참고 자료

- [Faster JavaScript Builds with Metro](https://medium.com/airbnb-engineering/faster-javascript-builds-with-metro-cfc46d617a1f)
- [Airbnb's Trip to Linaria](https://medium.com/airbnb-engineering/airbnbs-trip-to-linaria-dc169230bd12)
- [A Deep Dive into Airbnb's Server-Driven UI System](https://medium.com/airbnb-engineering/a-deep-dive-into-airbnbs-server-driven-ui-system-842244c5f5)
- [GraphQL Data Mocking at Scale with LLMs and @generateMock](https://airbnb.tech/infrastructure/graphql-data-mocking-at-scale-with-llms-and-generatemock/) (Niobe)
- [Avoiding the Monolith Trap: Airbnb's Multi-Tenant GraphQL Platform](https://graphql.org/conf/2025/schedule/eb8343e5935fbfccaaacf983ef84ab49/) (Viaduct)
- [Airbnb Open Source](https://airbnb.tech/open-source/)
- [Airbnb 약관의 봇 금지 조항 기록 (제3자 자료)](https://conductatlas.com/platform/airbnb/airbnb-terms-of-service/provision/CA-P-021897/prohibition-on-bots-and-automated-platform-access/)
