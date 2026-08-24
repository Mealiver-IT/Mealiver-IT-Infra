# 밀리버릿 (Mealiverit) — Infra

> 대규모 트래픽 선착순 쿠폰 발급 시스템의 인프라 레포지토리입니다.

이 레포는 애플리케이션 코드가 아니라, 애플리케이션이 돌아가는 **로컬 개발 인프라(Docker Compose)**와 **관찰가능성(Prometheus/Grafana)** 을 담당합니다. "재고 10,000장 vs 동시요청 20,000명 상황에서 초과발급 0건"을 검증하는 것이 이 프로젝트의 핵심 목표이기 때문에, 인프라 구성도 그 목표(정확성 검증)에 맞춰져 있습니다.

---

## 1. 전체 구성 한눈에 보기
[개발자 로컬 PC / 테스트 서버]
└─ docker-compose.yml
├─ mysql (로컬 개발/테스트용 DB)
├─ mysql-exporter (MySQL 내부 지표 → Prometheus)
├─ redis (중복요청 가드 + 재고 스냅샷 캐시, 단일 마스터)
├─ redis-exporter (Redis 서버 지표 → Prometheus)
├─ prometheus (지표 수집)
├─ grafana (대시보드)
├─ adminer (DB 웹 GUI)
├─ api (Mealiver-IT-BE 이미지, 로컬에서 mvnw로 대신 실행도 가능)
└─ fe (Mealiver-IT-FE 이미지, nginx가 /api/*를 api로 proxy_pass)

[원격 호스트] 100.125.247.64:3306 (Tailscale VPN 경유)
└─ MySQL — 실제 팀 공용 DB (더미데이터 적재 대상)


**왜 로컬 Docker Compose에도 MySQL이 있는데, 원격 DB(Tailscale)도 따로 쓰는가?**
- 로컬 `mysql` 컨테이너는 각자 PC에서 스키마/쿼리/배치를 빠르게 실험하고 깨뜨려도 되는 **개인 개발용 샌드박스**입니다.
- 원격 호스트(`100.125.247.64:3306`)는 팀 전체가 공유하는 **단일 정본(source of truth) DB**입니다. 정합성 검증처럼 팀 전체가 같은 결과를 봐야 하는 작업은 원격 DB 기준으로 수행합니다.
- 연결은 각자 공개 인터넷에 포트를 열지 않고 **Tailscale(WireGuard 기반 메시 VPN)**으로 사설망처럼 접속합니다.

---

## 2. 왜 이 인프라 구성을 선택했는가

### 2.1 Docker Compose
- **선정 이유**: 인프라도 "가볍게, 빨리 띄우고 내리는" 것이 중요해 `docker-compose.yml` 하나로 전체 스택을 명령어 한 줄(`docker compose up -d`)로 재현 가능하게 만드는 것을 우선했습니다.
- **담당 기능**: MySQL/Redis/observability(Prometheus·Grafana·exporter)/API/FE 컨테이너의 정의, 포트 매핑, 볼륨 마운트, 헬스체크.

### 2.2 MySQL
- **선정 이유**: 정합성 검증이 SQL 집계 쿼리로 이뤄지는 프로젝트 특성상, RDBMS의 강력한 제약조건(UNIQUE, 트랜잭션)이 "초과발급 0건, 1인 1매"를 물리적으로 보장하는 최후 방어선 역할을 합니다.
- **담당 기능**:
  - `uk_campaign_user` (campaign_id, user_id) UNIQUE — 1인 1매 최종 방어선
  - `uk_idempotency_key` UNIQUE — 중복 요청 방지
  - 더미데이터의 실제 저장소, 정합성 검증 SQL의 실행 대상
- **재고 동시성 제어**: 재고는 캠페인당 여러 개의 `campaign_stock_shard` 행(기본 50개, `app.stock-shard.count`로 설정 가능)으로 분산해 조건부 UPDATE(`remaining_stock > 0`)로 차감합니다. 초기에는 캠페인 단일 row(`campaign.remaining_stock`)를 직접 갱신해 hot-row 락 경합이 발생했었는데, 이 샤딩 구조로 해결했습니다. `campaign.remaining_stock`은 표시용 값으로 남아 15초 주기 재동기화 배치(`CampaignStockSnapshotReconciliationJob`)가 샤드 합계를 반영합니다.
- **`mysql-exporter`**: MySQL 내부 지표(Threads_running/connected, InnoDB 행 락 대기 등)를 Prometheus가 수집하도록 하는 사이드카. root가 아니라 전용 모니터링 계정(`exporter`, PROCESS/REPLICATION CLIENT/SELECT만)을 씁니다. `docker-entrypoint-initdb.d`는 볼륨이 이미 있으면 재실행 안 되므로, 새 환경이 아니면 이 계정을 한 번 수동으로 만들어야 합니다.

### 2.3 Redis
- **선정 이유**: DB 비관적 락 기반 재고 차감이 최종 정합성 판단이고, Redis는 그 앞단에서 두 가지 보조 역할만 합니다 — 재고 판단을 Redis가 직접 하지는 않습니다.
- **담당 기능**:
  - **중복요청 가드**(`CouponIssuanceDuplicateGuard`): 같은 (campaignId, userId) 조합의 요청이 이미 진행 중이면 캠페인 락을 아예 안 타고 즉시 거절 — `SETNX` + 최소 보유시간(MIN_HOLD 10초) + 처리 종료 시 명시적 `release()` 조합으로 동작. 부하테스트로 TTL-only/release-only 방식을 각각 실측하며 여러 차례 튜닝했습니다.
  - **재고 스냅샷 캐시**(`CampaignStockCache`): 확실한 품절만 미리 걸러내는 사전 필터(TTL 60초). Redis가 죽어도 예외를 삼키고 DB 폴백 경로로 넘어가도록 설계되어 있어, Redis 장애가 발급 정확성에 영향을 주지 않습니다.
- **중요한 운영 원칙**: Redis는 반드시 **마스터 단일노드**로만 구성합니다. 복제 노드를 참조하면 초과발급 위험이 생길 수 있어, Compose 설정에서 복제본을 두지 않는 것이 원칙입니다.

### 2.4 Prometheus + Grafana + Exporter
- **선정 이유**: 부하테스트 중 병목이 Tomcat/HikariCP/MySQL/Redis 중 어디에 있는지 실시간으로 봐야 했습니다(InnoDB 행 락 대기 폭증을 이 지표로 실측 진단한 사례가 있습니다).
- **담당 기능**: `mealiver-api`(Actuator `/actuator/prometheus`), `redis-exporter`, `mysql-exporter` 세 타겟을 5초 주기로 스크레이핑, Grafana에서 Tomcat 스레드/HikariCP/InnoDB 락 대기/Redis 명령 처리량 등을 대시보드로 확인.

### 2.5 Tailscale (원격 DB 접속)
- **선정 이유**: 팀원들이 하나의 공용 DB(`100.125.247.64:3306`)에 접속해야 하는데, 공개 인터넷에 MySQL 포트를 직접 여는 건 보안 위험이 크고, VPN 서버를 직접 운영하는 것도 오버엔지니어링입니다. Tailscale은 WireGuard 기반으로 설정 부담 없이 사설망처럼 묶어줍니다.
- **담당 기능**: 팀원 로컬 PC ↔ 원격 MySQL 호스트 간 암호화된 프라이빗 네트워크 연결.

### 2.6 k6 (부하테스트)
- **선정 이유**: 부하테스트 도구는 자유이며, k6는 JavaScript로 시나리오를 작성할 수 있어 팀 내 러닝커브가 낮고 `shared-iterations`/`ramping-vus` executor로 결정론적인 시나리오를 고정하기 쉽습니다.
- **담당 기능**: 재고 10,000 vs 동시요청 20,000, 동일 유저 다회 재요청(1인 1매) 등 시나리오 실행. 테스트 종료 후 최종 판정은 항상 DB 쿼리 기준입니다.
- ⚠️ **현재 스크립트가 이 레포에 커밋되어 있지 않습니다** — 실행 시 사용하는 `.js` 파일들의 위치를 확인해서 이 레포(`src/test/loadtest/` 등)에 추가하거나, 별도 관리 위치를 여기 명시해야 합니다.

---

## 3. 실행 방법

```bash
# 1. 인프라 전체 기동 (mysql, mysql-exporter, redis, redis-exporter, prometheus, grafana, adminer, api, fe)
docker compose up -d

# 2. API 이미지가 아직 없다면 인프라만 먼저 띄우고 API는 로컬(IDE/mvnw)에서 직접 실행
docker compose up -d mysql redis prometheus grafana adminer

# 3. 상태 확인
docker compose ps

# 4. 종료
docker compose down
mysql-exporter 최초 1회 설정 (기존 mysql_data 볼륨에 계정이 없다면):

CREATE USER 'exporter'@'%' IDENTIFIED BY '<비밀번호>';
GRANT PROCESS, REPLICATION CLIENT ON *.* TO 'exporter'@'%';
GRANT SELECT ON performance_schema.* TO 'exporter'@'%';
FLUSH PRIVILEGES;
mysql-exporter/.my.cnf에 위 계정 정보를 채워서 마운트합니다. ⚠️ 지금 이 파일이 평문 비밀번호가 든 채로 git에 커밋되어 있습니다 — 별도로 정리가 필요합니다(4절 참고).

4. 환경변수
DB 자격증명은 절대 application.properties나 docker-compose.yml에 하드코딩하지 않습니다. .env 파일(커밋 금지)로 관리합니다. ⚠️ 현재 .gitignore에 .env가 등록되어 있지 않고, mysql-exporter/.my.cnf는 실제로 커밋되어 있습니다 — 둘 다 정리가 필요합니다.

변수	설명	기본값
MYSQL_ROOT_PASSWORD	MySQL root 비밀번호	root1234
MYSQL_DATABASE	기본 스키마명	mealiver
MYSQL_USER / MYSQL_PASSWORD	앱 접속용 DB 계정	mealiver / mealiver1234
API_IMAGE	api 서비스가 pull할 이미지	ghcr.io/mealiver-it/mealiver-it-be:latest
FE_IMAGE	fe 서비스가 pull할 이미지	ghcr.io/mealiver-it/mealiver-it-fe:latest
GRAFANA_ADMIN_USER / GRAFANA_ADMIN_PASSWORD	Grafana 관리자 계정	admin / admin
5. 디렉토리 구조 (인프라 레포 기준)
.
├── docker-compose.yml         # 전체 서비스 정의
├── mysql-exporter/
│   └── .my.cnf                 # mysqld_exporter 접속 정보
├── prometheus/
│   └── prometheus.yml          # 스크레이프 타겟(api/redis-exporter/mysql-exporter)
├── .gitignore
└── README.md
6. 알려진 리스크 및 대응 (인프라 관점)
리스크	영향	대응
mysql-exporter/.my.cnf에 DB 비밀번호가 평문으로 커밋됨	자격증명 유출	비밀번호 교체 + git 추적 제외 필요 (미해결)
Redis를 복제 구성으로 잘못 띄우면 카운터/캐시 불일치 발생 가능	초과발급 재현 위험	Compose에서 Redis는 단일 마스터 노드로만 구성, 복제본 추가 금지
재고 단일 row 직접 갱신 시 hot-row 락 경합	대량 동시 발급 시 InnoDB 락 대기 폭증	campaign_stock_shard로 재고를 여러 행에 분산, 표시용 컬럼은 배치로 지연 동기화
Spring 프록시 self-invocation으로 @Transactional이 무시되는 경우	스케줄 배치가 매번 조용히 실패, DB 값이 stale해짐	프록시를 타는 진입 메서드(스케줄 트리거 메서드)에 트랜잭션을 직접 걸거나 별도 빈으로 분리
대량 동시 요청 시 HikariCP 커넥션 풀 고갈	재고 롤백까지 실패해 재고가 영구 유실될 수 있음	롤백 호출을 별도로 감싸 실패해도 원래 흐름은 유지하고 ERROR 로그로 수동 확인 유도
mysqld_exporter 최신 버전이 DATA_SOURCE_NAME 환경변수 방식을 제거함	잘못 설정 시 exporter가 재시작 루프에 빠짐(up=0)	.my.cnf 파일 마운트 + --config.my-cnf 플래그 방식 사용
OS의 net.core.somaxconn이 Tomcat accept-count보다 낮으면 accept 백로그가 캡핑됨	부하 초반 대량 연결 실패(dial: i/o timeout)	api 컨테이너에 sysctls: net.core.somaxconn=5000으로 Tomcat 설정과 맞춤
원격 호스트 네트워크 지연(Tailscale VPN 경유)	HikariCP 커넥션 타임아웃	connection-timeout, validation-timeout을 로컬 기준보다 여유 있게 설정
7. 관련 문서
⚠️ 아래 문서들은 현재 이 레포에서 파일을 찾을 수 없습니다. Notion 등 다른 곳에 있다면 링크를 이 섹션에 업데이트해주세요.

01_기획서.md — 프로젝트 개요, 전제조건
02_아키텍처.md — 동시성 제어 전략 비교 및 채택 근거
03_시스템설계.md — 정합성 검증 SQL, 부하테스트 시나리오
06_일정과역할.md — Phase별 로드맵, 인프라 담당 역할
08_리서치요약_쿠폰동시성사례.md — Redis 활용 근거, 배민/올리브영 실사례
