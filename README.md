# 💓 Heartbit: High-Availability Crypto Exchange Service

**Heartbit**은 대규모 트래픽 환경에서 안정적인 데이터 처리를 지향하는 가상자산 거래소 프로젝트입니다. **Java 25**와 **PostgreSQL**을 기반으로 데이터 무결성을 확보하며, **Redis Pub/Sub**과 **Caching** 전략을 통해 초당 수만 건의 시세 데이터를 실시간으로 전파하는 시스템을 구축하고 있습니다.

---

## 🚀 Key Features

### 1. Real-time Market Data System
* **WebSocket Streaming:** STOMP 프로토콜을 활용하여 실시간 체결 내역 및 호가 데이터를 100ms 이내의 지연 시간으로 클라이언트에 전송합니다.
* **Redis Pub/Sub:** 분산 서버 환경에서 실시간 시세 데이터를 효율적으로 공유하고 전파하기 위해 Redis의 발행/구독 모델을 적용했습니다.

### 2. High-Reliability Trading Engine
* **PostgreSQL Persistence:** 거래소 시스템의 핵심인 자산 데이터의 정밀도와 신뢰성을 위해 PostgreSQL의 강력한 트랜잭션 및 인덱싱 기능을 활용합니다.
* **Efficient Order Processing:** 사용자의 매수/매도 주문 요청을 처리하고, 실시간 체결 이벤트를 생성하여 시스템 전반에 공유합니다.

### 3. Performance & Bottleneck Analysis
* **k6 Load Testing:** k6를 활용하여 시스템의 한계치(TPS, Latency)를 측정하고, 부하 상황에서의 성능 저하 요인을 정밀 분석합니다.
* **Connection Pool Optimization:** 테스트 중 발생한 `SQLTransientConnectionException` 등 HikariCP 고갈 문제를 분석하여 OSIV 설정 해제 및 트랜잭션 범위 최적화를 수행했습니다.

---

## 🏗️ Architecture & Data Flow



1.  **Order Flow:** 사용자 주문(HTTP POST) → 서비스 로직 검증 → PostgreSQL 트랜잭션 반영.
2.  **Event Propagation:** 체결 발생 시 이벤트를 생성하여 Redis Pub/Sub 채널로 발행.
3.  **Real-time Broadcast:** WebSocket 서버가 채널을 구독하여 접속 중인 클라이언트들에게 시세 전파.


---

## 🛠️ Tech Stack

* **Language:** Java 25
* **Framework:** Spring Boot 3.4.x, Spring Data JPA
* **Database:** PostgreSQL 16, Redis 7.2
* **Infrastructure:** AWS EC2, Docker
* **Monitoring & Testing:** k6, Grafana

---

## 📝 Troubleshooting & Insights

* **Issue:** 대량의 웹소켓 연결 및 조회 요청 시 `HikariPool-1 - Connection is not available` 발생.
* **Cause:** OSIV 활성화로 인해 뷰 렌더링 시점까지 커넥션이 유지되어 풀이 빠르게 고갈됨을 파악.
* **Solution:** OSIV를 끄고 트랜잭션 범위를 서비스 계층으로 한정하여 커넥션 반납 속도를 개선, 에러 발생률을 대폭 낮춤.
