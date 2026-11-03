# 나만의 AI 오피스 만들기: 하네스로 디스코드 AI 직원 운영하기 (OpenClaw)

SSAFY 15기 코치세션(김예린 코치) 정리. 모델에게 일을 시키는 방식이 프롬프트, 컨텍스트, 하네스(Harness) 순으로 넘어왔고, 에이전트는 LLM에 하네스를 붙인 것이다. 서버에 OpenClaw 게이트웨이를 올려 디스코드 봇을 AI 직원으로 쓰는 과정을 따라가며 익혔다.

## 에이전트 = LLM + 하네스
하네스는 말(LLM)에 씌우는 마구(馬具)처럼, 모델이 일하도록 감싸는 실행 환경이다(보충). 세션 예시는 디스코드로 날씨 브리핑, 기업분석, TIL 회고(KPT)를 받는 구성이었다. 사용자가 보내는 요청은 4단계로 처리된다: 수신 → 라우팅(어느 에이전트에게 줄지) → 컨텍스트 조립(워크스페이스 마크다운을 시스템 프롬프트로) → 응답과 날짜별 memory 기록.

## 서버에 게이트웨이 올리기
Node 22 저장소를 먼저 등록해야 한다. 기본 저장소 버전으로는 동작하지 않았고, 22.22.3 이상인지 확인한다.

```bash
curl -fsSL https://rpm.nodesource.com/setup_22.x | sudo bash -
sudo dnf install -y nodejs && node -v
sudo npm install -g openclaw@latest && openclaw --version
openclaw plugins install discord --accept-capabilities
openclaw onboard --install-daemon --tui --gateway-bind loopback
sudo loginctl enable-linger $USER   # 24시간 구동. Linger=yes 확인
openclaw gateway status             # loopback, running, probe ok 확인
```

`--install-daemon`은 서비스 등록(재부팅 후 자동 시작), `--gateway-bind loopback`은 로컬에서만 접속되게 한다.

## 디스코드 봇 연결: 토큰은 .env, 변경은 dry-run 먼저
Developer Portal에서 앱을 만들고 공개 봇을 끈 뒤 Message Content Intent를 켠다. OAuth2 스코프는 `bot`, `applications.commands`, 권한은 채널 보기·메시지 보내기·스레드·임베드·첨부·기록 보기 여섯 개다. 토큰은 생성 시 한 번만 보이므로 바로 복사한다.

```bash
nano ~/.openclaw/.env            # DISCORD_BOT_TOKEN=<봇_토큰>
chmod 600 ~/.openclaw/.env       # 나만 읽고 쓰기
openclaw config patch --file ~/discord.patch.json5 --dry-run   # 검증
openclaw config patch --file ~/discord.patch.json5             # 적용
systemctl --user restart openclaw-gateway
```

```json5
{ channels: { discord: { enabled: true,
    token: { source: "env", id: "DISCORD_BOT_TOKEN" } } } }
```

접속 확인은 `journalctl --user -u openclaw-gateway.service -n 40 | grep -i discord`로 한다. 이후 `openclaw pairing list discord`, `openclaw pairing approve discord <코드>`로 내 계정을 승인한다. 접근 제한은 `dmPolicy: "allowlist"`, `allowFrom`, 서버별 `requireMention: true`, `users`로 건다.

## 에이전트 폴더 구조
`~/.openclaw/workspaces/{briefing,research,retro}/` 아래에 IDENTITY.md, SOUL.md, AGENTS.md, USER.md, memory/를 둔다. 에이전트의 성격과 규칙은 이 마크다운으로 정의한다.

## 하네스 엔지니어링의 한계
세션은 레거시화, Silent Failure, 과도한 토큰 사용을 한계로 꼽았다(슬라이드 제목만 있고 설명은 없어 내용은 미확인). 결론 슬라이드는 "문제를 해결하기 위해 어떤 기술을 써야 할까?"를 먼저 묻는다.

## 내 작업에 적용
- TIL 정리나 회고(KPT)를 디스코드 봇으로 받는 구조를 SSAFY 프로젝트의 에이전트 설계 참고로 쓸 수 있다.
- 에이전트 프롬프트를 IDENTITY/SOUL/AGENTS/USER 같은 파일로 나누는 방식은 프롬프트 엔지니어링 때 역할과 규칙을 분리하는 데 쓸 수 있다.

## 참고
- 260910목_코치세션_나만의 AI 오피스 만들기_김예린.pdf
