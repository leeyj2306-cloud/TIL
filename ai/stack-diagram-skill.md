# 스택 한 문장으로 아키텍처 다이어그램 그리기 (stack-diagram 스킬)

기술 스택을 문장으로 설명하면 공식 로고가 들어간 아키텍처 다이어그램을 만들어 주는 에이전트 스킬(MIT). 로고를 일일이 찾아 붙이고 화살표를 맞추는 수작업 없이, 설명 한 줄에서 공유 가능한 도면 한 장까지 이어진다.

## 입력은 스택을 설명하는 문장 하나
Claude Code, Codex, Cursor 등 스킬을 지원하는 에이전트에서 쓴다. Claude 앱에서는 Releases의 `stack-diagram.zip`을 스킬로 업로드한다. README의 예시 프롬프트:

```bash
npx skills add caru-ini/stack-diagram
```

> Diagram our stack: Next.js and a Hono API on Vercel. Postgres on Neon, auth with Clerk, payments with Stripe. GitHub Actions deploys.

클라우드 서비스뿐 아니라 모노레포 도구(pnpm, Turborepo, Biome 등)나 RAG 챗봇 같은 AI 앱 구성도 예시로 든다.

## 로고와 선 처리가 품질을 만든다
- 로고는 svgl → devicon → gilbarbara/logos → 브랜드 공식 저장소 순으로 받아 오고, 재색칠하거나 다시 그리지 않는다.
- 화살표가 텍스트를 가로지르지 않게 라벨을 피해 경로를 잡는다. 에이전트가 결과를 직접 렌더링해 확인한 뒤 넘긴다.
- 라이트/다크 테마를 모두 렌더링한다. 로고 타일은 흰색으로 유지해 브랜드 색이 그대로 보인다.

## 출력 형식과 요구 사항
- 기본은 폰트와 로고가 내장된 단일 HTML 파일로, 네트워크 없이도 렌더링된다. Claude 아티팩트로 게시하거나 레포·위키에 넣을 수 있다.
- PNG, 의존성 없는 React 컴포넌트로도 뽑을 수 있다. PNG는 Playwright + Chromium을 쓴다.
- Python 3.9+가 필요하고, 로고를 받으려면 네트워크가 되어야 한다. 로고는 스킬에 번들되지 않고 빌드 시점에 내려받는다.
- 로고와 브랜드명은 각 소유자의 상표이며, 다이어그램에 쓴다고 보증을 뜻하지 않는다.

## 내 작업에 적용
- 팀 프로젝트 README, 발표 자료, 포트폴리오에 들어갈 아키텍처 그림을 스택 설명 한 줄로 초안까지 만들 수 있다. (보충) 직접 쓸 때는 서비스별 연결(프로토콜, 호출 방향)까지 문장에 적어야 화살표 라벨이 정확해질 것이다.
- 포트폴리오에는 HTML 대신 PNG로 내보내 쓰는 편이 다루기 쉽다. (보충)

## 참고
- [caru-ini/stack-diagram (GitHub)](https://github.com/caru-ini/stack-diagram)
