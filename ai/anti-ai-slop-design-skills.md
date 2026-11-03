# AI스러운 UI를 막는 디자인 스킬 4종 비교 (Anti-AI-slop Design Skills)

AI가 만든 화면은 보라색 그라데이션, Inter 폰트, 가운데 정렬 히어로, 똑같은 카드 그리드로 수렴한다. 이를 막으려고 코딩 에이전트에 디자인 판단 기준을 주입하는 스킬이 나와 있고, 네 가지는 접근이 서로 다르다.

## UI UX Pro Max: 선택지를 주는 카탈로그
`nextlevelbuilder/ui-ux-pro-max-skill`(MIT). 스타일, 색상 팔레트, 폰트 조합, UX 가이드라인을 데이터베이스로 갖고 있고, 제품 유형(핀테크, 헬스케어 등)에 맞는 스타일과 안티패턴을 검색해 디자인 시스템 초안을 만든다. 예: 은행 서비스에는 "AI 보라/핑크 그라데이션"을 쓰지 말 것. 빈 화면에서 출발할 때 방향을 잡아 주는 쪽이다.

```bash
/plugin marketplace add nextlevelbuilder/ui-ux-pro-max-skill
/plugin install ui-ux-pro-max@ui-ux-pro-max-skill
```

## Impeccable: 나쁜 패턴을 걸러내는 필터
Paul Bakaus 제작(Apache 2.0). Anthropic 공식 `frontend-design` 스킬을 확장해, 타이포·색(OKLCH)·공간·모션·인터랙션·반응형·UX 라이팅 7개 레퍼런스와 슬래시 명령(버전에 따라 18~20개), "하지 말 것" 목록(Inter·Roboto 같은 기본 폰트 등)을 담았다. 생성보다 검수·개선에 강하다.

```bash
npx skills add pbakaus/impeccable
/teach-impeccable        # 프로젝트 디자인 맥락을 한 번 저장
/impeccable audit        # 기술 품질 점검
/impeccable critique     # 위계·명확성 등 UX 리뷰
```

## Taste Skill: 다이얼로 강도 조절
`Leonxlnx/taste-skill`(MIT). SKILL.md 상단의 숫자(1~10)로 레이아웃 변주, 모션 강도, 시각 밀도를 조절한다. 프레임워크에 묶이지 않고, `taste-skill`, `soft-skill`, `minimalist-skill`처럼 취향별 스킬이 나뉘어 있다.

```bash
npx skills add https://github.com/Leonxlnx/taste-skill --skill "taste-skill"
```

## Hallmark: 거부 규칙과 자체 검사
Together AI의 Hassan El Mghari(Nutlope) 제작(MIT). 동사 네 개로 쓴다: 기본(새 UI 생성), `audit`(코드를 안티패턴 기준으로 채점만 하고 수정 안 함), `redesign`(카피와 정보 구조는 유지하고 재설계), `study`(스크린샷·URL에서 디자인 DNA를 뽑아 `design.md`로 내보냄). 생성 후 "슬롭 테스트"로 스스로 검사하고, `.hallmark/log.json`에 이전 결과를 기록해 같은 프로젝트에서 구조가 반복되지 않게 한다.

```bash
npx skills add nutlope/hallmark
```

## 한눈에 비교
| 스킬 | 역할 | 잘 맞는 상황 |
|---|---|---|
| UI UX Pro Max | 스타일·팔레트·폰트 제안 | 빈 화면에서 방향 정하기 |
| Impeccable | 안티패턴 필터, 검수 명령 | 이미 만든 UI 다듬기 |
| Taste Skill | 강도 다이얼 | 분위기·밀도를 숫자로 조절 |
| Hallmark | 거부 규칙 + audit/redesign/study | 기존 코드 진단, 레퍼런스 분석 |

## 내 작업에 적용
- 식물 나눔 같은 와이어프레임 프로토타입이나 포트폴리오 페이지를 만들 때, UI UX Pro Max로 방향을 잡고 Impeccable이나 Hallmark의 `audit`로 AI 티가 나는 부분을 걸러내는 조합을 시도해 볼 만하다. (보충: 한 리뷰 글이 이 조합을 권장한다.)
- 팀에 이미 디자인 토큰이 있으면 UI UX Pro Max의 카탈로그가 오히려 충돌할 수 있다. 이때는 audit 용도로만 쓴다.
- 스타 수, 규칙 개수, 명령 개수는 출처와 버전마다 달라 기재하지 않았다. 설치 전 각 저장소에서 확인한다.

## 참고
- [nextlevelbuilder/ui-ux-pro-max-skill](https://github.com/nextlevelbuilder/ui-ux-pro-max-skill)
- [pbakaus/impeccable](https://github.com/pbakaus/impeccable)
- [Leonxlnx/taste-skill](https://github.com/Leonxlnx/taste-skill)
- [Nutlope/hallmark](https://github.com/Nutlope/hallmark)
