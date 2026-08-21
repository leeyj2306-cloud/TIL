# [기획/백엔드] 실시간 비공개 경매(Vickrey) 플랫폼 BidLive 기술 설계 및 부하 테스트 트러블슈팅 정리

오늘 진행한 **BidLive(실시간 비크리 라이브 경매 플랫폼)** 프로젝트의 아키텍처 의사결정, 동시성 및 부하 테스트(k6) 환경 구축, 그리고 개발 중 직면했던 트러블슈팅 과정을 정리해 봅니다.

---

## 1. 왜 WebRTC와 비크리(Vickrey) 경매인가? (기획 배경 및 기술적 당위성)

### ① 일반 스트리밍(HLS) vs WebRTC 초저지연 미디어 파이프라인

* **문제 정의**: 기존 라이브 커머스 스트리밍 방식(HLS/DASH)은 최소 5초에서 최대 30초의 버퍼링 및 지연(Latency)이 발생합니다. 경매 마감 직전 초 단위의 가격 경쟁 상황에서 방송 화면과 입찰 서버의 시차가 발생하면 경매 정합성이 완전히 무너집니다.
* **해결책 (WebRTC SFU 채널 분리)**:
* **미디어 채널 (WebRTC SFU)**: 판매자(1명)와 다수 시청자(N명) 간의 영상 송수신을 0.5초 미만(Sub-second Latency)의 초저지연으로 중계하여 시차 없는 실물 검증을 보장합니다.


* **상태/거래 채널 (WebSocket/STOMP)**: 영상 트래픽의 변동이 가격 입찰 데이터에 영향을 주지 않도록 네트워크 파이프라인을 물리적으로 분리했습니다.



### ② 비공개 2등가 경매(Vickrey Auction)의 경제학적 타당성

* **메커니즘 원리**: 입찰자는 각자 자신이 지불 가능한 최대 금액을 비공개로 적어내고, 최종 낙찰자는 **최고가 입찰자**가 되되 실제 지불 금액은 **2등 입찰가(Second-Price)** 또는 하한가(Reserve Price)를 결제합니다.
* **도입 효과**: 1등가 경매 특유의 눈치 싸움과 과열 입찰(승자의 저주)을 완화하며, 자신이 생각하는 진정한 가치(True Valuation)를 그대로 적어내는 진실 유도형(Truthful Bidding) 거래를 만듭니다.

---

## 2. 백엔드 핵심 아키텍처 및 동시성 제어

### ① Redis Lua Script 기반 In-Memory 단일 원자 연산

경매 마감 직전 초 단위 동시 입찰 트래픽 스파이크 시 발생할 수 있는 Race Condition과 DB Lock 경합(Deadlock)을 방지하기 위해 입찰 처리 경로를 이원화했습니다.

* **Redis 원자적 처리 (단일 스레드 실행)**:
1. 하한가 및 본인 기존 입찰가 초과(상향) 검증
2. 포인트 지갑 잔액 검증 및 차액 선-홀드(에스크로) 처리
3. 입찰 기록 및 타임스탬프 갱신


* **PostgreSQL 영속화 트랜잭션**:
경매 종료 시점에 Redis 상태와 원장(PointLedger)을 대조하여 이중 검증 후 최종 낙찰 및 차액 복구를 안전하게 반영합니다.

```
[Client 동시 입찰 요청] 
       │
       ▼
[Spring Boot (API Gateway / Controller)]
       │
       ▼ (단일 트랜잭션 원자성 보장)
[Redis Lua Script Engine] ──> 입찰 자격/잔액 검증 ➔ 포인트 차액 홀드 ➔ 입찰 순위 갱신
       │
       ▼ (비동기 이벤트 발행)
[PostgreSQL Ledger DB] ───> 최종 마감 시 거래 원장 및 정산 이중 검증 기록

```

---

## 3. 부하 테스트(k6) 환경 구축 및 실전 트러블슈팅

### 🚨 트러블슈팅 1: Spring Boot 기동 시 JWT Secret Base64 디코딩 오류

* **증상**: `./gradlew bootRun` 실행 중 `io.jsonwebtoken.io.DecodingException: Illegal base64 character: '_'` 에러 발생과 함께 서버 기동 실패.
* **원인 분석**: `application.yml` 또는 `.env`에 등록된 JWT Secret Key 문자열 내에 표준 Base64 규격에 어긋나는 특수문자(`_` 등)가 포함되어 디코딩 파싱 실패.
* **해결 방법**: 표준 Base64 규격(A-Z, a-z, 0-9, +, /)을 준수하는 256비트 이상의 유효한 시크릿 키 문자열로 교체 및 환경변수 주입.

---

### 🚨 트러블슈팅 2: 로컬 서버 구동 포트 충돌 (`Port 8080 was already in use`)

* **증상**: `Web server failed to start. Port 8080 was already in use.`
* **원인 분석**: 이전에 백그라운드에서 강제 종료되지 않고 잔류한 Java 프로세스가 8080 포트를 지속 점유함.
* **해결 방법**:
* Git Bash / 터미널에서 포트 점유 프로세스 PID 강제 종료:
```bash
taskkill //F //PID $(netstat -ano | grep :8080 | awk '{print $5}' | head -n 1)

```





---

### 🚨 트러블슈팅 3: K6 중단 잔여 데이터로 인한 Redis-DB 정합성 충돌 (500 에러)

* **증상**: K6 부하 테스트 스윕(Sweep) 중 `BidCommandService.verifyStoredHistory()` 단계에서 500 에러 발생.
* **원인 분석**: 이전 K6 부하 테스트가 비정상 중단되면서 PostgreSQL의 입찰 테이블(4,214건)과 Redis의 입찰 카운트(2,875건) 상태 간 싱크가 어긋남. 백엔드의 데이터 정합성 검증 가드레일이 이를 탐지하여 예외를 발생시킴.
* **해결 방법**:
1. 부하 테스트용 더미 테이블 초기화:
```sql
TRUNCATE trade_point_ledger, bid, auction_participant RESTART IDENTITY CASCADE;

```


2. 로컬 테스트 계정 지갑 잔액 원복 및 Redis의 `bid:{bid}:auction:1:state` 관련 해시 키 초기화 후 K6 재실행.



---

## 4. 학습 회고 (Retrospective)

* **정직한 아키텍처 문서화**: 미검증된 가설 수치를 무작정 포트폴리오에 적는 것보다, **"시스템의 병목 원인을 정확히 진단하고(HikariCP 풀, 웹소켓 브로커), 이를 해결하기 위한 구체적인 튜닝 아키텍처를 설계하는 논리적 과정"** 자체가 엔지니어링의 핵심 역량임을 배웠습니다.
* **로컬 벤치마크 환경의 중요성**: 클라우드 인스턴스가 닫혀도 로컬 Docker 환경과 k6 스크립트를 경량화하여 직접 재현하고 Before/After 수치를 증명하는 테스트 파이프라인의 필요성을 체감했습니다.