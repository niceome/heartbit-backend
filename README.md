# Heartbit

> 대량 거래 주문을 견뎌낼 수 있는 가상 화폐 거래소 플랫폼

<br>

## 📋 프로젝트 개요

Heartbit은 실제 거래소와 동일한 흐름으로 매수/매도 주문을 경험할 수 있는 모의 투자 거래 플랫폼입니다.

주문 → 매칭 → 체결 → 자산 정산의 전 과정을 실시간으로 처리하며, 캔들 차트 시각화, 종목별 채팅, WebSocket 기반 실시간 시세/체결 스트리밍을 제공합니다.

핵심 설계 원칙은 **매칭은 단일 스레드로 직렬화하고, 후처리(알림·DB 저장·브로드캐스트)만 병렬화하여 락 없이 처리량을 확보**하는 것입니다.

<br>

## 📦 기술 스택

| 분류 | 기술 |
|------|------|
| Language / Framework | Java 25, Spring Boot 4.0.1, Spring Security + JWT, Spring Data JPA |
| Concurrency | LMAX Disruptor 4.0 (RingBuffer 기반 단일 스레드 매칭 엔진) |
| Realtime | WebSocket (STOMP) |
| Database | PostgreSQL 17, Flyway (마이그레이션), HikariCP (커넥션 풀) |
| Cache / Lock | Redis 7 (Lettuce), Redisson 3.17 |
| Monitoring | Spring Actuator, Micrometer, Prometheus, Grafana |
| Test / Quality | JaCoCo (커버리지) |
| Infrastructure | Docker Compose (Redis / PostgreSQL / Prometheus / Grafana 로컬 구성) |

<br>

## 🏗️ 아키텍처 특징

DDD 스타일로 **Order / Trade / Asset / Invest / Notification / Chatroom** 도메인을 분리하고, 그 중심에 Disruptor 기반 단일 스레드 매칭 엔진을 둔 구조입니다.

한때는 종목(`categoryId`)별로 담당 서버를 나누는 샤딩형 MSA(`OrderRouter`, Redis `order-sharding-channel`)까지 구현했으나, 운영 복잡도 대비 이득이 적어 `remove msa` 커밋으로 단일 모놀리스 구조로 되돌렸습니다.

### 실시간 데이터 처리 흐름

```
주문 요청
    → OrderEventProducer (Disruptor RingBuffer, buffer=4096 적재)
    → MatchingHandler (단일 스레드) — 카테고리별 OrderBook에서 매칭
    → 병렬 소비
        ├── OrderEventHandler  (WebSocket 브로드캐스트)
        └── OrderDbHandler     (DB 반영)
    → 체결 결과 100건 또는 배치 종료 시점마다 그룹핑
    → TradeMarketBroadcaster
        → /topic/ticker, /topic/trades, /topic/charts, /topic/orderbook/lastPrice
```

매칭을 단일 스레드로 직렬화하여 이중 체결 위험을 원천 차단하고, 후처리만 병렬로 소비하는 것이 핵심 설계입니다.

<br>

## 🔧 트러블슈팅

### 1. 동시성 매칭 — 분산락 대신 Disruptor 선택

다수 사용자가 동시에 주문을 넣을 때 인메모리 호가창(`OrderBook`)이 레이스 컨디션에 노출됐습니다.

Redisson을 도입했지만 매칭 로직 자체에는 분산락을 걸지 않고, 대신 Disruptor의 RingBuffer(`buffer=4096`)로 모든 주문 이벤트를 순차 처리해 동시 접근을 원천 차단하는 방식을 택했습니다. 분산락은 자산 잔고 갱신처럼 DB row 단위 경합에만 남기고, 주문 매칭 자체는 락 없는 구조로 재설계했습니다.

### 2. 자산 정합성 — 비관적 락

매수 시 주문가능금액을 미리 차감(`blockCash`)하고 체결 시 확정(`settleBuyTrade`)하는 2단계 구조에서 이중 차감/음수 잔고 위험이 있었습니다.

`findByMemberIdWithLock`에 `PESSIMISTIC_WRITE` 락을 걸어 회원 자산 row를 `SELECT ... FOR UPDATE`로 잠그는 방식으로 해결했습니다(단일 DB 구조라 분산락 없이도 직렬화 보장). 이 과정에서 `OrderBook.add()`의 null 처리 누락, 서버 재시작 복구 로직의 트랜잭션 누락으로 인한 NPE도 함께 수정했습니다.

### 3. OSIV(Open Session In View) 비활성화

`open-in-view: false`로 전환 후, 트랜잭션 밖에서 지연로딩이 발생하던 `TradeService.getTradeList()`에 명시적으로 `@Transactional(readOnly = true)`를 추가했습니다.

커넥션을 요청 전체 동안 붙잡지 않아 풀 고갈 위험은 줄었지만, 서비스 메서드마다 트랜잭션 경계를 명확히 잡아줘야 하는 트레이드오프가 있었습니다.

### 4. HikariCP 커넥션 풀 확장

`maximum-pool-size`를 20 → 30으로 상향했습니다. 봇 트래픽 시뮬레이터로 부하를 걸어보며 풀 부족으로 인한 대기/타임아웃을 관찰한 뒤 조정했습니다.

<br>

## ⚡ 성능 개선

### Redis 캐싱 도입 (현재가 조회, 실측)

현재가 조회에 캐시-어사이드 패턴(TTL 60초)을 적용했습니다. 부하 테스트 결과(1,000 요청 기준):

| 지표 | DB 직접 조회 | Redis 캐시 적용 | 개선폭 |
|------|--------------|-----------------|--------|
| 평균 응답시간 | 5.81ms | 2.99ms | 약 48% ↓ |
| p95 응답시간 | 8.34ms | 6.12ms | 약 27% ↓ |
| 처리량 (TPS) | 166.98 req/s | 318.75 req/s | 약 1.9배 ↑ |

### 벌크 Insert (JDBC 배치)

체결/투자내역 저장을 JPA 단건 저장 대신 `JdbcTemplate.batchUpdate` + `PreparedStatement.addBatch()`로 전환했습니다.

FK 객체 참조(`Invest.trade`)를 `Long tradeId`로 바꿔 배치 저장 시 불필요한 연관관계 조회를 없앴고, 체결 이벤트도 100건 단위 또는 배치 종료 시점마다 그룹핑해 처리함으로써 체결 폭주 상황에서 DB 쓰기 횟수를 줄였습니다.

<br>

## 🔐 자산 정산과 동시성 보장

매수 접수 시 `blockCash`(선차감) → 체결 시 `settleBuyTrade` / `settleSellTrade`(확정 정산)의 2단계 구조이며, 각 단계는 `PESSIMISTIC_WRITE` 락 안에서 수행됩니다.

봇 트래픽처럼 초당 수십 건이 몰려도 잔고 음수화나 이중 차감을 DB 트랜잭션 레벨에서 차단합니다. 매칭 엔진 자체는 Disruptor 단일 스레드라 이중 체결 위험이 없고, 자산 갱신만 별도로 락을 거는 **이원화된 정합성 전략**을 취하고 있습니다.

<br>

## 📈 차트(캔들) 데이터 처리

별도 OHLC 집계 테이블 없이, `trade` 테이블에서 커서 기반 페이지네이션으로 최근 체결을 가져와 애플리케이션 레벨에서 분 단위로 그룹핑해 시가/고가/저가/종가를 계산합니다(최초 접속 시 빈 차트 방지용 REST 폴백).

실시간 갱신은 `ConcurrentHashMap`으로 "현재 분" 캔들 상태를 들고 있다가 체결마다 갱신하여, DB 재조회 없이 인메모리에서 STOMP로 흘려보냅니다.

<br>

## 🧩 다중 서버 확장 시도와 흔적

프로덕션에 서버 2대(`app-1`, `app-2`) 운영을 전제로 종목별 샤딩(`OrderRouter`)과 8채널 Redis Pub/Sub 릴레이(`ws-ticker` / `trades` / `charts` / `orderbook` / `invest` / `chat` / `notification-channel` 등, `RedisMessageListenerContainer` 기반)까지 구현했습니다.

각 인스턴스가 자신에게 연결된 WebSocket 클라이언트에게 이벤트를 릴레이할 수 있는 구조였으나, 최종적으로 단일 인스턴스 운영으로 방향을 정리하며 이 구독 인프라를 통째로 삭제하고 `SimpMessagingTemplate` 직접 브로드캐스트로 되돌렸습니다.

> ⚠️ **알려진 이슈**: 위 롤백이 불완전하여, 현재 `NotificationService.send()`는 Redis로 발행만 하고 이를 받아 WebSocket으로 릴레이할 구독자가 존재하지 않는 상태로 남아 있습니다(채팅 경로는 정상 복구됨). 리팩터링 과정에서 발생한 누락으로, 향후 정리 대상입니다.

<br>

## 🛠️ 로컬 실행

```bash
# Redis / PostgreSQL / Prometheus / Grafana 기동
docker compose up -d

# 애플리케이션 실행
./gradlew bootRun
```
