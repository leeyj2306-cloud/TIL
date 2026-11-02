# REA: 에이전트로 앱·바이너리 동작을 역공학하는 MCP (Reverse Engineer Anything)

소스 코드 없이 앱의 기능이 어떻게 동작하는지 에이전트가 직접 조사하게 해 주는 오픈소스 MCP 서버 겸 CLI. 분석 결과를 근거(evidence)와 한계까지 함께 돌려줘서, 에이전트가 설명하고 구현·테스트하는 단계로 이어 가기 쉽다.

## 구조: 에이전트가 MCP로 호출하고, 분석은 로컬에서
에이전트가 MCP로 대상을 조사하면 REA가 코드와 참조, 알 수 없는 부분(unknowns)을 근거와 함께 반환한다. 에이전트는 이를 바탕으로 추가 질문을 하거나 구현한다. 분석은 로컬에서 실행되고, 에이전트의 모델 제공자에게는 도구 결과만 전달된다. CLI도 같은 워크플로를 쓴다.

## 설치와 기본 사용
Node.js와 npm이 필요하다(README 기준 22.x 이상 등 버전 조건 있음). `setup`이 에이전트(Claude Code, Codex, Cursor, Gemini CLI 등)에 MCP 서버와 워크플로 지침을 등록하며, 기존 설정은 백업하고 변경 내용을 승인받은 뒤 적용한다. 설치 후 에이전트를 재시작한다.

```bash
npx rea-agents setup                      # 에이전트에 MCP + 지침 등록
npm install --global rea-agents           # rea 명령 설치
rea update                                # 업데이트
npx -y rea-agents@latest analyze-javascript-application /absolute/path/to/app --json
```

에이전트에게는 이렇게 요청한다: "Notes 앱의 검색이 어떻게 동작하는지 근거와 함께 이해하고, 내 프로젝트에 비슷한 기능을 만들어줘."

## 분석 대상은 넓고, 필요한 도구는 대상마다 다르다
JavaScript/Electron 앱, .NET 어셈블리, 웹사이트, HAR 같은 네트워크 캡처는 별도 네이티브 엔진 없이 정적 분석이 가능하다. 네이티브 바이너리의 의사코드·어셈블리는 Hopper, Ghidra, IDA, Kuna 중 하나가 필요하다. 그 밖에 Android APK(JADX와 JDK 필요), 펌웨어, 패킷 캡처, SQLite, 모델 파일(GGUF/safetensors) 헤더 등도 지원한다.

## 녹화한 API 세션으로 RL 환경 만들기
웹 앱 세션의 요청·응답 기록(HAR)을 에이전트가 호출할 도구로 바꿔 실행 가능한 환경으로 내보낸다. 에이전트가 원본 앱 없이 폴더 생성·삭제 같은 작업을 연습하거나 평가받게 할 수 있고, 녹화로 확인된 변경만 지원한다. 실행에는 AgentEnv와 Docker가 필요하다.

```bash
rea environment-build ./session.har ./my-env --name my-env --json
agent-env run ./my-env
```

## 사용 범위
합법적인 연구, 교육, 상호운용성, 소프트웨어 보존, 본인 소유 또는 분석 권한이 있는 소프트웨어에만 쓴다. 상용 도구(IDA, Hopper, JEB)의 라이선스는 별도로 합법 취득해야 한다. 라이선스는 MIT.

## 내 작업에 적용
- MCP 도구 설계 참고: 결과에 근거와 한계를 함께 반환하는 방식과, 입력·출력 스키마를 compact/full 프로필로 나눠 컨텍스트를 아끼는 방식(`REA_MCP_INPUT_SCHEMA_PROFILE`)을 AI 에이전트·API 설계에 가져올 수 있다.
- 다른 서비스의 기능을 벤치마킹할 때는 권한이 있는 대상인지부터 확인한다.

## 참고
- [morluto/rea (GitHub)](https://github.com/morluto/rea)
