# Anthropic Scion: 에이전트 규모 코딩을 위해 Git을 대체한 사내 VCS (Version Control System)

AI 에이전트가 코드를 쓰는 양이 폭증하면 병목은 코딩이 아니라 Git·CI 같은 개발 인프라로 옮겨간다. Anthropic은 이 문제를 사내 전용 VCS인 Scion으로 풀었고, 일반 팀에는 "작게 격리된 저장소 + Git" 구성이 여전히 현실적인 답이다.

## 왜 Git이 병목이 됐나
- 일일 머지가 수만 건, CI 체크아웃이 수천만 회 규모로 늘면서 Git 미러 인프라가 버티지 못했다.
- (보충) 보도 기준 하루 머지 수는 2026년 5월 약 2,500건 → 9월 3만 건 이상으로 늘었다.
- 7명 팀이 5개월 만에 Scion을 만들고 모노레포 전체를 이전했다.

## Scion의 구조: 분산 복제 대신 중앙 서버 스트리밍
- Git은 오프라인 작업을 위해 저장소 전체를 로컬에 복제한다. 에이전트는 항상 온라인이므로 이 전제가 낭비가 된다.
- Scion은 중앙 서버 API로 필요한 파일·트리만 받아 오는 희소(sparse) 체크아웃을 쓴다.
- `git worktree` 대신 서버 기반 Changeset과 캐시 데몬(`scion-daemon`)으로 작업 공간을 만든다.
- 결과: 인프라 비용(CPU·메모리) 약 1/200, 에이전트 작업 공간 생성 7.1분 → 5.1초.

## 모노레포인데 에이전트에겐 폴리레포처럼
- 시판 에이전트 개발 환경(ADE, 예: Cursor·Cline)은 로컬 파일시스템과 `git worktree`에 의존한다.
- 모노레포가 커지면 워크트리 생성 I/O, 로컬 인덱싱(LSP, ripgrep), 리베이스가 병목이 되어 에이전트가 멈춘다.
- Scion은 조직 입장에선 단일 컨텍스트의 모노레포를 유지하면서, 에이전트에겐 필요한 서브트리만 내려줘 "초경량 폴리레포"처럼 보이게 가상화했다.

## 일반 팀에 주는 시사점
- Scion은 사내 전용이고 오픈소스·상용 출시 계획이 없다. Anthropic도 범용 Git 대체재가 아니라고 밝혔다.
- 자체 인프라·전담 팀이 없는 환경에서는 Git을 유지하고, 에이전트별 작업 범위를 작은 저장소·작은 경로로 격리하는 편이 안전하다.
- (보충) Anthropic은 GitHub·GitLab 등과 협력해 Scion의 아이디어(서버 측 RPC 등)를 Git 생태계로 가져오고 싶다고 했다. "폴리레포가 정답"은 자료 작성자의 실무 결론이고, 장기적으로 Git 쪽이 바뀔 여지는 있다.
- (보충) 이름이 같은 Google Scion(컨테이너 기반 멀티 에이전트 실험 도구, Git worktree 사용)과는 무관하다.

```bash
# 에이전트 작업 범위를 좁히는 Git 기본기: 필요한 경로만 체크아웃
git clone --filter=blob:none --sparse <repo-url>
cd <repo>
git sparse-checkout set backend/ai-api
git worktree add ../agent-a -b feat/agent-a
```

## 내 작업에 적용
- SSAFY 팀 프로젝트에서 에이전트를 병렬로 돌릴 때 FE/BE/AI 영역별로 저장소나 디렉터리 경계를 나누고, 에이전트마다 별도 브랜치·워크트리를 주면 충돌과 컨텍스트 낭비를 줄일 수 있다.
- 인프라·플랫폼 직무 면접에서 "AI 시대에 병목이 개발 인프라로 이동한다"는 사례로 쓸 만하다.

## 참고
- AI_코딩으로_늘어난_변경량에_대응해_Git을_자체_버전_관리_시스템_Scion으로_교체.pdf
- [Anthropic built its own git replacement for AI-scale coding (The New Stack)](https://thenewstack.io/anthropic-scion-git-replacement/)
- Anthropic 엔지니어링 블로그, "Redesigning source control at Anthropic" (2026-10-13)
