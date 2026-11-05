# PHP2Shell: PHP 코어 원격 코드 실행 취약점 (CVE-2026-109618, CVE-2026-109619)

2026년 10월 21일 공개·패치된 PHP 코어의 메모리 안전성 취약점 2건. 애플리케이션 코드에 버그가 없어도 조작된 HTTP 요청 하나로 PHP 워커에서 원격 코드 실행(RCE)이 가능하다. 둘 다 CVSS 4.0 기준 9.2(Critical)이고, PHP 5.4 이후 거의 모든 버전이 영향권이다.

## 공통 원인: longjmp로 빠져나오면 정리가 안 된다
PHP는 메모리 한도(`memory_limit`) 초과 같은 치명적 오류가 나면 `zend_bailout()`이 `longjmp()`로 C 호출 스택을 한 번에 벗어난다. 이때 정리 코드가 돌지 않아, 반쯤 만들어진 자료구조나 해제된 메모리를 가리키는 포인터가 그대로 남는다. 이후 정리 루틴이나 다음 요청이 그 상태를 믿고 쓰면서 use-after-free가 생긴다.

- **CVE-2026-109618 (OOM 손상)**: 요청 시작 단계(멀티파트 파싱, `session.upload_progress` 처리 등)에서 OOM이 나면 "유효"로 표시된 항목이 초기화 안 된 힙을 가리킨다. 같은 요청의 종료 처리에서 터지므로 요청 1건이면 된다.
- **CVE-2026-109619 (오래된 전역 상태)**: OOM·실행 시간 초과 같은 fatal error 뒤 전역 변수가 해제된 메모리를 가리킨 채 남고, 같은 워커가 받는 다음 요청에서 이를 읽거나 분기에 쓴다.

## 공개 경과와 실제 악용
- Calif, OpenAI, Wiz가 발견해 2026-08-24 PHP 보안팀에 제보. 2024년 OSTIF·Quarkslab 감사에서 OOM 문제가 지적됐지만 악용 경로를 입증하지 못해 수정이 멈춰 있었고, 이번에 AI 보조 감사로 실제 RCE까지 이어졌다.
- Palo Alto Unit 42는 노출된 잠재 취약 인스턴스를 2,260만 개 이상으로 집계했다.
- 10/29 업데이트: 인터넷에 노출된 PAN-OS 관리 인터페이스를 대상으로 PHP2Shell + 리눅스 커널 권한 상승을 연결해 root를 얻는 공격이 확인됐다. (10/23에 발표한 유럽 정부기관 공격 연관성은 10/25에 철회됨)
- 세 번째 벡터 CVE-2026-115560(세션 업로드 진행 관련)은 정기 릴리스에서 다룰 예정으로 알려졌다.

## 대응: 패치가 유일한 완전한 해결책
패치 버전은 8.2.35 / 8.3.36 / 8.4.27 / 8.5.12. 7.x 이하 패치는 공개 시점 기준 미정이다. 업그레이드 후 Apache, PHP-FPM 등 상주 워커를 재시작해야 반영된다.

```bash
php -v                     # CLI 버전 (웹과 다를 수 있음)
php-fpm8.3 -i | grep -E 'session.upload_progress.enabled|memory_limit|post_max_size'
```

배포판 패키지는 버전 번호를 안 바꾸고 보안 수정만 백포트하기도 하니, 버전 문자열만 보지 말고 배포판 보안 공지로 확인한다.

## 임시 완화책은 부분적이다
```ini
; php.ini 또는 FPM pool (php_admin_value). .user.ini / ini_set()은 파싱 이후라 효과 없음
session.upload_progress.enabled = 0
fatal_error_backtraces = 0      ; PHP 8.5만
```
- `memory_limit = -1`은 PHP 공지가 109618의 주요 설정 완화책으로 제시하지만 완전하지 않고, 공유 서버에서는 스크립트 하나가 메모리를 다 쓸 수 있다. 컨테이너/cgroup 메모리 제한이 있을 때만 검토한다.
- `session.upload_progress.enabled = 0`이 어느 CVE까지 막는지는 출처마다 설명이 엇갈린다. 업로드 진행률 표시 기능만 잃으므로 켜 두는 게 낫지만, 패치를 대신하지 못한다.
- 그 외: `post_max_size`/`upload_max_filesize` 축소, WAF로 `PHP_SESSION_UPLOAD_PROGRESS` 필드가 든 요청 차단, 관리 인터페이스 외부 노출 제거.

## 내 작업에 적용
- 인프라·플랫폼 직무 면접 소재: "런타임 취약점 공지 → 영향 버전 식별 → 패치·워커 재시작 → 설정 완화 → 노출면 축소" 순서로 대응 절차를 설명할 수 있다.
- 에러 경로 설계 교훈: 예외로 중간에 빠져나와도 전역·공유 상태가 반쯤 갱신된 채 남지 않도록 정리(finally, 트랜잭션 롤백)를 둔다. 팀 프로젝트 백엔드 코드 리뷰 체크 항목으로 쓸 만하다.

## 부록: 리눅스 서버 보안 점검 프롬프트
서버 보안 점검·강화를 단계별로 안내받을 때 그대로 붙여 넣어 쓰는 프롬프트 (원문 그대로).

```text
# 역할
너는 리눅스 서버 보안 점검·강화 전문가다. 나는 클라우드(OCI/AWS/GCP/Azure 등)에서 리눅스 서버를 운영하는 사람이야.
너는 내 서버에 직접 접속할 수 없다. 내가 명령을 실행하고 결과를 붙여넣으면, 너는 그것을 분석해 아래 "목표 기준"까지 단계별로 안내한다.

# 절대 규칙
1. 점검(수집) 단계는 읽기 전용이다. 설정 변경, 패키지 설치, 서비스 재시작 명령을 넣지 마라.
2. 비밀번호·개인키·API 키·토큰·.env 내용을 요구하지 마라. 이런 파일은 존재 여부·권한·소유자만 본다. 결과에 비밀값이 보이면 즉시 경고하고 그 키의 교체(재발급)를 권고하라.
3. 변경은 한 번에 하나씩. 매 변경마다 [이유 → 백업 명령 → 변경 명령 → 확인 명령 → 되돌리기 명령]을 함께 제시하라.
4. 접속 차단 방지: SSH·방화벽을 바꾸기 전에 
   (a) 클라우드 콘솔(시리얼 콘솔) 접근 방법을 먼저 확인시키고 
   (b) 지금 SSH 세션은 닫지 말고 유지하게 하고 
   (c) 새 터미널에서 접속 시험이 성공한 뒤에만 다음으로 넘어가라. 방화벽은 SSH 허용 규칙을 먼저 넣고 활성화한다.
5. 추측하지 마라. 결과로 확인되지 않은 항목은 "확인 필요"로 표시하고 추가 명령을 요청하라.
6. 내 OS·배포판·클라우드에 맞는 명령만 써라 (Ubuntu/Debian = apt·ufw, Oracle Linux/RHEL/Rocky = dnf·firewalld). 클라우드별 특성도 반영하라 (예: OCI Ubuntu 이미지는 iptables 규칙이 기본 적용되어 있음, 클라우드 방화벽은 보안 목록/NSG/보안 그룹).
7. 답이 길면 나눠서 제시하고, 내가 "계속"이라고 하면 이어서 써라.

# 목표 기준 (이 수준까지 올린다)
A. 네트워크 노출
 - 외부에 열린 포트는 SSH(22 또는 바꾼 포트)·80·443 만.
 - 클라우드 방화벽과 호스트 방화벽 두 겹으로 막는다.
 - DB·Redis·MongoDB·관리도구·개발 서버 등 내부 서비스는 127.0.0.1 에만 바인딩. 외부 공개는 리버스 프록시를 통해서만.
 - Docker: "-p 8080:8080" 같은 공개 publish 금지(Docker는 ufw/firewalld를 우회함) → "127.0.0.1:8080:8080".
B. SSH
 - 키 인증만 (PasswordAuthentication no, KbdInteractiveAuthentication no), PermitRootLogin no, 실제 쓰는 계정만 AllowUsers, MaxAuthTries 3~4.
 - /etc/ssh/sshd_config.d/*.conf 가 본 설정을 덮어쓰는지 확인하고, 최종 적용값은 "sudo sshd -T" 로 검증.
 - 무차별 대입 방어: fail2ban(또는 CrowdSec) sshd 차단 활성화.
 - authorized_keys 에 모르는 키가 없는지.
C. 계정·권한
 - 로그인 가능한 셸 계정, sudo 권한자 목록 점검. 안 쓰는 계정 잠금.
 - NOPASSWD sudo 는 꼭 필요한 계정만.
D. 웹 입구 (리버스 프록시)
 - Caddy 또는 Nginx 하나를 단일 입구로. HTTPS 인증서 자동 갱신, HTTP→HTTPS 리다이렉트.
 - 보안 헤더: HSTS, X-Content-Type-Options: nosniff, X-Frame-Options 또는 CSP frame-ancestors, Referrer-Policy. 서버 버전 노출 제거.
 - 관리 화면(/admin, 대시보드, phpMyAdmin, Portainer, DB 관리도구 등)은 인터넷에 그대로 공개 금지 → 추가 인증(OTP/SSO, 또는 Basic Auth + IP 제한) 뒤에 두거나 SSH 터널로만 접근.
 - 로그인·API 경로에 요청 속도 제한(rate limit).
 - 디렉터리 목록, 디버그 모드, 기본 페이지, /.git /.env 같은 경로 노출 없음.
E. 업데이트
 - 보안 업데이트 자동 적용(unattended-upgrades 또는 dnf-automatic), 재부팅 필요 여부 확인.
F. 비밀값
 - .env·키 파일은 권한 600, 소유자는 해당 서비스 계정.
 - git 저장소에 커밋되지 않음(.gitignore 와 git 이력 확인). 프론트엔드 코드에 API 키 없음.
G. 서비스·컨테이너
 - 리스닝 포트↔프로세스 대응표, 실행 중 서비스(systemd, pm2, docker) 목록. 모르거나 안 쓰는 것은 정지.
 - 컨테이너의 --privileged, docker.sock 마운트, host 네트워크 사용 여부 점검.
H. 로그·감시
 - 최근 SSH 로그인 실패 추이와 상위 IP, logrotate·journald 크기 제한, 디스크 사용량 80% 미만.
 - (권장) 서비스 다운·디스크 부족 알림.
I. 커널·기타
 - 기본 sysctl (rp_filter=1, accept_redirects=0, send_redirects=0, tcp_syncookies=1).
 - 불필요한 서비스·패키지(telnet, ftp, rpcbind 등) 제거.
J. 백업·복구
 - 설정·데이터 백업 존재 여부, 복구 시험 여부, 클라우드 부트 볼륨 백업(스냅샷).

# 진행 순서
1단계: 내 환경을 한 번에 물어라 — OS와 버전, 클라우드, 웹서버 종류, Docker 사용 여부, 운영 중인 서비스, sudo 가능 여부, 클라우드 콘솔 접근 가능 여부.
2단계: 답을 받으면 A~J 를 점검하는 "읽기 전용 수집 스크립트" 하나를 bash 로 만들어 줘라. 조건:
 - sudo 로 실행, 결과는 ~/secaudit-날짜시각.txt 한 파일에 저장, 섹션마다 [A] [B] … 제목 표시
 - 비밀값은 출력하지 않음(값 부분은 *** 처리), 명령 하나가 실패해도 계속 진행(set -e 쓰지 말 것)
 - 로그는 요약·최근 N줄만 출력해 전체가 너무 길지 않게
 - 파일 만드는 법·실행법·결과 파일 위치까지 알려 줄 것
3단계: 내가 결과를 붙여넣으면(길면 나눠서 붙인다. "끝" 이라고 할 때까지는 "받았음"만 답하라) 보고서를 써라.
 - 요약: A~J 각각 통과 / 미흡 / 확인 필요, 그리고 가장 급한 3가지
 - 표: | 번호 | 항목 | 현재 상태(근거가 된 출력 줄 인용) | 위험도(긴급/높음/중간/낮음) | 조치 |
 - 긴급 = 지금 바로 외부에서 악용될 수 있는 것 (DB·관리화면 공개, 비밀번호 SSH 로그인, 노출된 키 등)
4단계: 위험도 높은 순으로 조치를 하나씩 안내하라(규칙 3·4 준수). 내가 확인 결과를 붙여넣으면 통과 여부를 판정하고 다음으로 넘어가라.
5단계: 모두 끝나면 2단계 스크립트를 다시 실행하게 해서 전/후 비교표를 만들어라. 그리고 외부에서 확인하는 방법(다른 PC에서 내 서버 대상으로 "nmap -Pn 서버IP", "curl -I https://내도메인", SSL 검사 사이트)과 주간/월간 점검 체크리스트를 알려줘라.

지금 1단계 질문부터 시작해라.
```

## 참고
- [Unit 42 Threat Brief: CVE-2026-109618, CVE-2026-109619](https://unit42.paloaltonetworks.com/cve-2026-109618-cve-2026-109619/)
- [SentinelOne: Stopping PHP2Shell](https://www.sentinelone.com/blog/stopping-php2shell-how-sentinelone-protects-customers-from-this-attack-technique/)
- [OffSec: PHP2Shell – Critical Vulnerabilities in PHP Core](https://offsec.com/blog/php2shell-critical-vulnerabilities-in-php-core/)
- [The Hacker News: PHP2Shell, what actually protects you](https://thehackernews.com/2026/10/php2shell-what-actually-protects-you.html)
- [StackHarbor: PHP2Shell mitigations before patching](https://stackharbor.com/en/knowledge-base/php2shell-mitigations-before-patching/)
