# ego lite — AI 에이전트를 위한 에이전트 브라우저

> 2026-10-20 TIL

- 사이트: https://lite.ego.app/ko
- GitHub: https://github.com/citrolabs/ego-lite
- 만든 곳: Citro Labs
- 가격: 무료 (클라우드 비용 없음)
- 이번에 **Windows용도 배포**됨

## 한 줄 요약

Claude Code, Codex, Cursor 같은 코딩 에이전트가 **내가 쓰던 Chrome 로그인 상태 그대로** 웹 자동화를 돌릴 수 있게 해주는 Chromium 기반 브라우저.

## 핵심 동작 방식: "명령 실행 방식"

기존 aside가 사람이 손으로 하던 조작을 옆에서 대신해 주는 형태였다면, ego는 에이전트가 **명령을 코드로 묶어서 한 번에 실행**하는 구조다.

1. 에이전트가 페이지를 **시맨틱 스냅샷(Snapshot)** 으로 읽는다. (HTML 원문보다 토큰 소모가 적고, 클래스명이 바뀌어도 `@N` 참조가 유지됨)
2. `snapshot / click / fill / navigate / wait / capture` 같은 도구를 조합한 **JavaScript 코드 묶음**을 작성한다.
3. ego-browser가 그 묶음을 페이지에서 **한 번에 실행**한다.

→ Playwright나 Puppeteer처럼 "도구 한 번 호출 → 응답 대기 → 다음 동작 결정"을 반복하는 왕복(round-trip)이 없어서 빠르고 토큰도 덜 쓴다. 그래서 aside보다 가볍고 **코딩 에이전트 특화**라고 볼 수 있다.

## 구성 요소

| 구성 | 역할 |
| --- | --- |
| **ego(lite)** | Chrome처럼 쓰는 Chromium 기반 브라우저 본체 |
| **ego-browser** | ego(lite)를 에이전트 브라우저로 바꿔주는 **스킬**. 온보딩 때 에이전트의 스킬 디렉터리에 설치됨 |
| **Space** | 브라우저 안의 에이전트 전용 독립 작업 공간 |

사용은 온보딩 후 에이전트에 `/ego-browser` 명령을 입력하고 원하는 작업을 말하면 끝. SDK 연동도, 원격 디버깅 설정도 필요 없다.

## 특징

- **로그인 상태 상속**: 온보딩에서 한 번 클릭으로 Chrome의 탭·북마크·비밀번호·확장 프로그램·쿠키·프로필을 가져옴. 에이전트도 같은 로그인 세션을 그대로 씀 (Gmail, Notion, LinkedIn, 사내 도구 재인증 불필요)
- **Space 분리**: 에이전트는 자기 Space에서만 작업해서 내 탭·마우스·포커스를 뺏지 않음. 원하면 Space로 전환해서 에이전트가 일하는 걸 지켜볼 수도 있음
- **병렬 실행**: 서로 다른 Space에서 작업을 개수 제한 없이 동시에 실행 (공식 소개로는 100개 이상)
- **깊은 접근**: 페이지에 스크립트를 주입하지 않고 자체 Chromium 엔진 내부에서 스냅샷을 생성 → 교차 출처 iframe, shadow DOM, Stripe/Salesforce/Intercom 같은 서드파티 위젯 내부까지 접근 가능
- **프라이버시**: 방문 기록·쿠키·비밀번호·읽은 페이지 내용 모두 로컬에만 저장. 설정 과정에서 기록하는 건 "Chrome 데이터 이전을 선택했는지 여부" 하나뿐
- **호환 에이전트**: 셸 명령을 실행할 수 있는 에이전트면 모두 가능 (Claude Code, Codex, Cursor, Kiro, Continue, Gemini CLI, Hermes Agent, OpenClaw, OpenCode, 커스텀 에이전트 등)
- **일상 브라우저로도 사용 가능**: 주소창, 탭 그룹, 북마크, 확장 프로그램, 시크릿 모드 등 Chrome과 똑같이 동작

## 벤치마크 (공식 발표, Real-World Bench 기준)

| 도구 | 작업당 평균 시간 | 완전 성공률 | 작업당 평균 비용 |
| --- | --- | --- | --- |
| **ego-browser** | **398s** | **93.5%** | **$1.64** |
| chrome-devtools-cli | 587s | 61.3% | $4.95 |
| agent-browser | 605s | 62.9% | $2.66 |
| browser-harness | 615s | 77.4% | $2.43 |
| playwright-cli | 648s | 71.0% | $3.42 |

> 벤더가 직접 공개한 수치라서, 참고용으로만 보는 게 좋겠다.

## 다른 도구와의 차이

- **Playwright / Puppeteer**: CSS 선택자 기반이라 DOM 구조가 바뀌면 깨짐. 헤드리스 브라우저라 내 로그인 상태도 없음 → ego는 시맨틱 스냅샷의 `@N` 참조 + 실제 로그인 브라우저
- **Browser Use**: 자체 LLM 루프를 내장하고 별도 브라우저 인스턴스를 새로 띄움 → ego는 에이전트 종속 없이 내 브라우저를 그대로 사용
- **agent-browser (Vercel)**: 셸 명령으로 제어하는 CLI지만 자체 브라우저가 없고 내 세션을 공유하지 못함
- **브라우저 확장 프로그램 (Claude in Chrome 등)**: 특정 에이전트에만 묶이고, 내 브라우저를 가로채며, 병렬 실행이 어려움
- **클라우드 브라우저 (Browserbase 등)**: 로그인 상태가 없고 유료

## 활용 예

- **개발**: 이미 로그인된 브라우저에서 바로 E2E 테스트 (선택자·로그인 스크립트 관리 불필요)
- **리서치**: Notion, Confluence 같은 로그인 필요한 페이지까지 읽고 출처 정리
- **영업·마케팅**: 로그인해야 보이는 데이터(Sales Navigator, 소셜 미디어 등) 수집·분석
- **재무**: 여러 사이트에 로그인해 청구서 수집 (승인·결제는 사람이 직접)

## 느낀 점 / 메모

- "에이전트용 별도 브라우저"가 아니라 **"나와 에이전트가 같이 쓰는 하나의 브라우저"** 라는 컨셉이 핵심
- 로그인 세션을 에이전트가 그대로 물려받는 만큼, 어떤 계정·사이트에 접근을 허용할지는 신경 써서 써야 할 듯
- 재사용 가능한 스킬 기능은 아직 **출시 예정**

## 참고

- [ego(lite) 공식 사이트](https://lite.ego.app/ko)
- [문서 (Quick Start)](https://lite.ego.app/document/ko/docs/quick-start)
- [업데이트 노트](https://lite.ego.app/ko/changelog)
