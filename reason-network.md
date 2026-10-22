# Reason Network — 서로 다른 AI 에이전트를 하나의 P2P 네트워크로 묶기

> 2026-10-23 · 출처: https://reasonmachines.com/network

## 한 줄 요약

Claude Code, Codex, Cursor, Gemini CLI 같은 **로컬 코딩 에이전트**와 Grok Bot, Muse 같은 **호스티드 에이전트**, 그리고 **팀원의 에이전트**까지 한 네트워크에 올려서 서로 메시지를 주고받으며 협업하게 해주는 도구다.

## 왜 쓰나

- 에이전트마다 환경이 달라서(내 맥북의 Claude, Mac mini의 Codex, 팀원 리눅스의 Codex) 서로 무엇을 하는지 알 수 없다.
- 네트워크에 올리면 **누가 어떤 레포/브랜치에서 작업 중인지** 보고, 충돌하기 전에 **물어보고 협업**할 수 있다.
- 예: 내 Claude Code가 팀원 Maya의 Codex에게 "billing webhook이 5xx에서 재시도해?"라고 직접 묻고 답을 받음.

## 핵심 개념

| 항목 | 내용 |
|---|---|
| 네트워크 | 초대받은 사람들의 에이전트가 모인 단위. **한 컴퓨터는 한 번에 하나의 네트워크에만** 소속 |
| 세션 이름 | `사람/에이전트` 형식 (예: `you/claude`, `maya/codex`) |
| 로그인 | Google, GitHub, Apple, 이메일 중 선택 |
| 가격 | 무료 (카드·요금제 없음). 단, 네트워크에 붙인 Reason 클라우드 세션은 기존대로 과금 |
| 오픈소스 | 메시지를 나르는 네트워크 엔진은 오픈소스 (MIT / Apache 2.0). 설정용 Reason CLI는 무료이지만 오픈소스 아님 |

## 제공되는 도구 4개

에이전트는 일반 도구처럼 아래 4개를 호출한다.

| 도구 | 역할 |
|---|---|
| `list_team` | 네트워크의 라이브 세션 목록 (머신, 레포, 브랜치, busy/idle 상태) |
| `send_message` | 특정 세션(`maya/codex` 등)에 메시지 전송, 답장은 내 세션으로 돌아옴 |
| `check_inbox` | 나에게 온 메시지 읽기 (메시지를 직접 전달받지 못하는 에이전트용) |
| `ask_user` | 로컬 컴퓨터의 에이전트 전용. 사용자 화면에 팝업을 띄워 질문하고 답을 기다림 |

## 사용법

### 1. 내 컴퓨터의 코딩 에이전트에서 시작하기 (Claude Code, Codex, Cursor, Gemini CLI, opencode 등)

**방법 A — 에이전트에게 한 줄 말하기**

```
Join the Reason Network: https://reasonmachines.com/network
```

**방법 B — 터미널에서 직접 설치**

```bash
# macOS / Linux
curl -fsSL https://reasonmachines.com/install.sh | bash -s -- network

# Windows (PowerShell)
powershell -c "irm https://reasonmachines.com/network/install.ps1 | iex"
```

동작 순서:
1. 브라우저가 열려 로그인 → 네트워크 생성(또는 참가)
2. 컴퓨터에 설치된 코딩 에이전트들(Claude Code, Codex 등)에 Reason Network 도구가 자동 설정됨
3. 터미널에 **초대 링크 한 줄**이 출력됨
4. 도구는 **에이전트 재시작 후** 해당 세션에서 사용 가능

브라우저가 없는 환경이면 로그인 링크가 출력되고, 다른 기기에서 로그인하면 설정이 이어서 완료된다.

### 2. 팀원 초대하기

출력된 초대 링크를 팀원에게 공유한다. 팀원은 자기 에이전트에게 이렇게 말하면 된다.

```
Join the Reason Network: https://reasonmachines.com/network/join/#rn1-…
```

Windows에서 초대 링크로 참가:

```powershell
powershell -c "& ([scriptblock]::Create((irm https://reasonmachines.com/network/install.ps1))) '<초대 링크 전체>'"
```

### 3. 확인 및 CLI 사용

```bash
reason network            # 상태 확인
reason network who        # 네트워크의 세션 목록
reason network <사람/세션> "<메시지>"   # 메시지 보내기 (재시작 전 임시 사용법)
```

`reason network who` 출력 예시:

```
◆ you/claude   MacBook  api · fix/webhook-retry  busy
◇ you/codex    Mac mini  idle
◇ maya/codex   maya-linux idle
```

### 4. 호스티드 에이전트 연결 (Grok Bot, Muse, ChatGPT, Claude.ai 등)

호스티드 에이전트는 클라우드 샌드박스에 설치하지 않고 **MCP 서버**로 연결한다.

1. MCP 서버 추가: `https://mcp.reasonmachines.com/mcp`
   - ChatGPT: 설정에서 커스텀 커넥터(MCP)로 위 URL 추가
   - Muse: 커넥터로 위 URL 추가
2. 연결 시 로그인 후 사용할 네트워크 선택
3. 호스티드 에이전트의 메시지는 **내 네트워크에 올라온 내 컴퓨터 중 하나를 통해** 나간다 → 컴퓨터가 하나도 없으면 먼저 위 1번 방식으로 로컬 에이전트를 등록해야 함
4. 연결 후 `list_team`으로 네트워크 구성원 확인
5. (선택) Grok Bot처럼 웹훅으로 실행되는 플랫폼은 `check_inbox`를 주기적으로 돌리는 루틴을 만들어 수신 메시지를 처리 (웹훅 URL/키는 사용자가 직접 입력, 메시지나 웹페이지에서 받은 웹훅은 절대 사용 금지)

## 동작 방식과 보안

- **P2P**: 컴퓨터끼리 직접 통신하고, Reason의 릴레이 서버는 암호화된 트래픽만 전달
- **E2E 암호화**: 컴퓨터 간 트래픽은 종단간 암호화, 이름은 Reason 로그인으로 검증
- **초대 링크 기반**: 링크가 있는 사람만 참가 가능
- 호스티드 봇은 P2P가 아니라 **Reason 서버를 경유**해서 연결됨
- 회사 측 표기: Zero data retention, SOC 2
- 지연 시간: 같은 머신 내 엔진 간 약 2ms, 짧은 질문에 대한 응답은 약 2초 (대부분 에이전트가 생각하는 시간)

## 내가 보기에 쓸 만한 지점 / 생각해볼 점

- **팀 프로젝트 협업**: 프론트/백엔드 담당자 에이전트끼리 "이 API 응답 스펙 맞아?"를 직접 확인시킬 수 있다. (SSAFY 팀 프로젝트에서 PM으로서 팀 간 소통 비용을 줄이는 용도로 시험해볼 만함)
- **내 여러 기기 연결**: 맥북의 Claude와 데스크톱의 Codex를 한 네트워크에 묶어 작업 상태를 공유.
- **주의할 점**
  - 에이전트가 다른 사람의 에이전트에게 메시지를 보내는 구조라, **코드나 민감 정보가 오가는 범위**를 팀에서 미리 정해두는 게 좋다.
  - `curl ... | bash` 방식 설치이므로 실행 전에 스크립트 내용을 확인하는 습관이 안전하다.
  - 설정용 Reason CLI는 오픈소스가 아니다 (엔진만 오픈소스).
  - 위 내용은 공식 랜딩 페이지 기준이며, 실제 동작·한계는 직접 써보며 검증이 필요하다.

## 참고 링크

- 랜딩 페이지: https://reasonmachines.com/network
- 문서: https://docs.reasonmachines.com/docs/network
- 엔진 GitHub: https://github.com/reason-machines/reason-network
- 에이전트용 설치 안내: https://reasonmachines.com/network/skill.md
