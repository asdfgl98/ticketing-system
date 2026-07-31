# 기차 예매 시스템 아키텍처 및 학습 설계

## 1. 목적

이 프로젝트의 목적은 기차 예매 시스템을 직접 구현하고 실제 VPS에 배포하면서 대용량 트래픽과 분산 시스템의 핵심 개념을 경험하는 것이다.

최종 성능 목표는 다음과 같다.

- 동일 좌석에 동시 요청이 발생해도 중복 예매 0건
- 예매 API 100 RPS
- 예매 API 응답 시간 p95 500ms 이하
- Redis, Kafka 또는 worker 장애 후에도 데이터 정합성과 복구 가능성 유지

빠른 완성보다 문제를 직접 재현하고 해결 방법을 비교하는 것을 우선한다.

## 2. 기술 구성

- 메인 서버: NestJS
- 데이터베이스: PostgreSQL
- ORM: Prisma
- 캐시와 분산 조정: Redis
- 이벤트 스트리밍: Kafka
- 부하 테스트: k6
- 애플리케이션 로그: Winston
- 배포: Hetzner VPS, Dokploy, Docker, Nginx

Prisma는 일반 CRUD와 migration에 사용한다. 데이터베이스 잠금이나 작업 선점처럼 ORM만으로 이해하기 어려운 부분에서는 제한적으로 raw SQL을 사용한다.

## 3. 시스템 경계

```text
Client / k6
    |
  Nginx
    |
server x N
    |-- PostgreSQL
    |-- Redis
    `-- Kafka
          |-- reservation-worker x N
          `-- scheduler-worker x N
```

각 애플리케이션의 역할은 다음과 같다.

- `server`: HTTP API, 열차 조회, 좌석 점유, 예매와 가상 결제 요청
- `reservation-worker`: Kafka 이벤트를 통한 예매 및 가상 결제 후속 처리
- `scheduler-worker`: 만료 좌석 해제, 실패 작업 재처리, 스케줄 작업
- PostgreSQL: 예매 데이터 정합성의 최종 기준
- Redis: 캐시, rate limit, 임시 상태, 인스턴스 간 조정
- Kafka: 비동기 작업 전달과 장애 후 재처리

`server`와 worker는 독립된 NestJS 프로세스와 Docker 컨테이너다. 모든 서버 노드에 모든 worker를 함께 배치할 필요는 없다. HTTP 트래픽이 증가하면 `server`를, 이벤트 적체가 증가하면 worker를 독립적으로 확장한다.

초기에는 하나의 PostgreSQL 데이터베이스를 공유한다. 서비스별 데이터베이스 분리는 현재 학습 범위에 포함하지 않고, 코드 모듈과 쓰기 책임을 논리적으로 구분한다.

## 4. 구현 및 학습 단계

### 1단계: 기본 예매 시스템

NestJS, Prisma, PostgreSQL만 사용해 열차 조회, 좌석 점유, 가상 결제, 예매 확정과 취소 흐름을 완성한다. 이 단계에서는 Redis와 Kafka를 사용하지 않는다.

완료 기준:

- 단일 사용자의 전체 예매 흐름이 동작한다.
- 핵심 비즈니스 규칙에 단위 테스트가 있다.
- 실제 PostgreSQL을 사용하는 API 통합 테스트가 있다.

### 2단계: 데이터베이스 동시성

k6로 동일 좌석에 동시 요청을 보내 중복 처리 문제를 먼저 재현한다. 이후 PostgreSQL transaction, connection pool, row lock, 격리 수준과 제약 조건을 학습하고 적용한다.

완료 기준:

- 동일 좌석 동시 요청에서 중복 예매가 발생하지 않는다.
- 긴 transaction이 connection pool과 응답 시간에 미치는 영향을 설명할 수 있다.
- lock 대기와 deadlock을 로그 및 PostgreSQL 상태로 관찰할 수 있다.

### 3단계: Redis

`server`를 여러 인스턴스로 실행한 뒤 Redis 캐시, rate limit, 임시 점유 만료와 분산 조정을 단계적으로 추가한다.

Redis는 성능과 조정 계층이며 예매 정합성의 유일한 보루로 사용하지 않는다. Redis가 중단돼도 PostgreSQL 제약을 통해 중복 예매가 발생하지 않아야 한다.

완료 기준:

- 캐시 적용 전후의 조회 성능을 비교한다.
- 여러 `server` 인스턴스가 공통 상태를 공유한다.
- Redis 장애 시 시스템의 저하 방식과 DB 정합성을 확인한다.

### 4단계: Kafka와 worker

동기 흐름의 후속 작업을 Kafka 이벤트로 옮기고 `reservation-worker`, `scheduler-worker`를 분리한다.

다음 개념을 단순한 발행과 소비부터 순서대로 학습한다.

- producer, consumer, topic
- partition과 consumer group
- offset과 재처리
- at-least-once 전달과 멱등성
- 제한된 재시도와 DLQ
- DB 변경과 이벤트 발행을 연결하는 Outbox 패턴

완료 기준:

- worker를 여러 개 실행해도 하나의 작업 결과가 중복 반영되지 않는다.
- worker 종료 후 다른 consumer가 작업을 이어받는다.
- Kafka 중단 중 생성된 작업이 복구 후 처리된다.

### 5단계: 로깅과 부하 테스트

초기 로깅은 Winston만 사용한다.

```text
NestJS -> Winston JSON -> stdout -> docker logs
```

로그에는 필요에 따라 `requestId`, `reservationId`, `eventId`, `service`, `instanceId`, 처리 시간과 오류 stack을 포함한다. 결제 정보나 인증 정보 같은 민감 데이터는 기록하지 않는다.

Loki, Grafana, Grafana Alloy는 초기 범위에서 제외한다. 인스턴스와 worker가 늘어 `docker logs`만으로 추적하기 어려워진 뒤 중앙 로그 시스템으로 추가한다.

k6는 기능 테스트를 대신하지 않는다. 동시성 정합성, 처리량, 응답 시간과 오류율을 검증한다.

완료 기준:

- 한 요청이 `server`와 worker에서 처리되는 과정을 식별자로 추적할 수 있다.
- 100 RPS에서 중복 예매 0건을 유지한다.
- 예매 API p95 응답 시간이 500ms 이하이다.

### 6단계: 다중 VPS

첫 VPS에서는 구성요소를 Docker 컨테이너로 분리한다. 이는 프로세스 분리 학습 환경이지 인프라 고가용성 환경은 아니다.

이후 2~3대의 VPS로 확장한다.

```text
Node A: Nginx + server
Node B: server + reservation-worker
Node C: scheduler-worker + 선택한 데이터 인프라
```

첫 확장의 목표는 애플리케이션 수평 확장, 로드밸런싱, worker 장애 격리와 스케줄 작업 중복 방지다. PostgreSQL, Redis와 Kafka 자체의 완전한 고가용성은 별도의 후속 학습 범위로 둔다.

2코어, 4GB RAM VPS에서는 모든 서비스를 동시에 여유 있게 운영하기 어렵다. 각 단계에서 필요한 서비스만 먼저 실행하고 메모리 제한과 실제 사용량을 측정한다. swap은 장애 방지 보조 수단으로만 사용하며 정상 처리 용량으로 간주하지 않는다.

## 5. 테스트 전략

테스트는 종류보다 검증 목적을 기준으로 선택한다.

### 단위 테스트

- 비즈니스 규칙과 상태 변경을 빠르게 검증한다.
- DB, Redis와 Kafka는 mock 또는 port 인터페이스로 대체한다.
- NestJS 의존성 주입 구성이 중요한 경우 `TestingModule`을 사용한다.
- 단순 클래스는 NestJS 컨테이너 없이 직접 생성해 테스트할 수 있다.

### 통합 테스트

- 실제 PostgreSQL, Redis와 Kafka의 동작을 검증한다.
- transaction, lock, 제약 조건, 이벤트 발행과 소비는 mock 테스트만으로 대체하지 않는다.
- 각 인프라는 해당 학습 단계에서 테스트 환경에 추가한다.

### E2E 테스트

- 열차 조회부터 예매 확정과 취소까지 사용자 흐름을 API 경계에서 검증한다.
- 인증, validation, 예외 변환과 실제 DB 변경을 함께 확인한다.

### k6 테스트

- 같은 좌석에 요청이 집중되는 경쟁 시나리오를 검증한다.
- 일반적인 조회와 예매가 섞인 트래픽을 검증한다.
- RPS, p95, 오류율과 중복 예매 건수를 함께 확인한다.

커버리지 숫자 자체를 목표로 하지 않는다. 금전 또는 데이터 정합성에 영향을 주는 규칙, 동시 실행 시 결과가 달라지는 로직, 외부 시스템 실패와 재처리를 우선 테스트한다.

## 6. 장애 처리 원칙

- PostgreSQL 장애 시 예매를 성공으로 응답하지 않는다.
- Redis 장애 시 성능과 일부 조정 기능은 저하될 수 있지만 DB 정합성을 유지한다.
- Kafka 발행 실패 시 DB에 저장된 작업 정보를 기반으로 복구할 수 있어야 한다.
- worker 재시작과 Kafka 중복 전달을 정상 상황으로 간주하고 멱등하게 처리한다.
- 반복 실패 이벤트는 무한 재시도하지 않고 분리해 조사할 수 있게 한다.
- 로그 출력 장애가 핵심 예매 처리를 장시간 막지 않게 한다.

## 7. 의도적으로 미루는 범위

- 실제 PG 결제 연동
- 서비스마다 독립된 데이터베이스
- Kubernetes
- 다중 리전
- PostgreSQL, Redis, Kafka의 완전한 고가용성 클러스터
- 초기 단계의 Loki, Grafana, Grafana Alloy
- 처음부터 세분화된 마이크로서비스

이 항목들은 기본 시스템의 정합성과 장애 복구를 이해한 뒤 필요에 따라 확장한다.

## 8. 참고 문서

- [NestJS Microservices](https://docs.nestjs.com/microservices/basics)
- [NestJS Kafka](https://docs.nestjs.com/microservices/kafka)
- [PostgreSQL Explicit Locking](https://www.postgresql.org/docs/current/explicit-locking.html)
- [Prisma Connection Pool](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/databases-connections/connection-pool)
- [Redis Distributed Locks](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
- [Grafana k6 Thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/)
