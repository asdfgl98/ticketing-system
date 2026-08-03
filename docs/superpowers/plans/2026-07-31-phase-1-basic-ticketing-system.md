# Phase 1 Basic Ticketing System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** NestJS, Prisma, PostgreSQL만 사용해 단일 사용자의 열차 조회, 좌석 점유, 가상 결제, 예매 확정과 취소 흐름을 완성한다.

**Architecture:** 첫 단계는 NestJS 표준 모드의 모듈러 모놀리스로 구현한다. 기능별 모듈은 controller, service, repository 경계를 가지며 PostgreSQL을 단일 진실 공급원으로 사용한다. Redis, Kafka, DB lock과 수평 확장은 다음 단계에서 추가한다.

**Tech Stack:** Node.js 24 LTS, TypeScript, NestJS, Prisma ORM 7, PostgreSQL, Jest, Supertest, Docker Compose, npm

## Global Constraints

- 실행 애플리케이션 이름은 `server`다.
- ORM은 Prisma를 사용하고 PostgreSQL 연결에는 `@prisma/adapter-pg`를 사용한다.
- Redis, Kafka, DB row lock, 분산 락과 실제 PG 연동은 1단계 범위에서 제외한다.
- 데이터 정합성 규칙은 service와 PostgreSQL 제약 조건으로 표현한다.
- 각 작업은 실패하는 테스트 작성, 최소 구현, 테스트 통과, 커밋 순서로 진행한다.
- 초기 로그는 Winston JSON을 stdout에 출력한다. Loki, Grafana와 Alloy는 사용하지 않는다.

---

## Planned File Structure

```text
src/
├─ main.ts                         # NestJS bootstrap과 전역 validation
├─ app.module.ts                   # 최상위 모듈 조합
├─ health/                         # 프로세스 생존 확인 API
├─ database/                       # Prisma client 수명과 DB 연결
├─ train/                          # 열차 운행 및 좌석 조회
├─ seat-hold/                      # 좌석 임시 점유 규칙
├─ reservation/                    # 예매 확정과 취소
├─ payment/                        # 교체 가능한 가상 결제 port
└─ common/
   ├─ errors/                      # 도메인 오류와 HTTP 오류 변환
   └─ logging/                     # Winston JSON 설정
prisma/
├─ schema.prisma                   # PostgreSQL 데이터 모델
├─ migrations/                     # Prisma migration 결과
└─ seed.ts                         # 로컬 열차와 좌석 데이터
test/
├─ setup.ts                        # E2E 테스트 공통 준비
└─ booking-flow.e2e-spec.ts        # 전체 예매 흐름
docker-compose.yml                 # 개발용 PostgreSQL
prisma.config.ts                   # Prisma CLI 설정
.env.example                       # 환경 변수 예시
```

---

### Task 1: NestJS `server`와 Health API

**Files:**

- Create: `package.json`
- Create: `nest-cli.json`
- Create: `tsconfig.json`
- Create: `src/main.ts`
- Create: `src/app.module.ts`
- Create: `src/health/health.controller.ts`
- Test: `src/health/health.controller.spec.ts`
- Delete after scaffold: `src/app.controller.ts`
- Delete after scaffold: `src/app.controller.spec.ts`
- Delete after scaffold: `src/app.service.ts`

**Interfaces:**

- Produces: `GET /health -> { status: "ok" }`

- [ ] **Step 1: NestJS 표준 애플리케이션 생성**

Run:

```bash
npx @nestjs/cli@latest new server --directory . --package-manager npm --skip-git --strict
```

Expected: 루트에 `package.json`, `nest-cli.json`, `src/`, `test/`가 생성되고 기존 `docs/`는 유지된다.

- [ ] **Step 2: Health API의 실패 테스트 작성**

```typescript
import { Test } from "@nestjs/testing";
import { HealthController } from "./health.controller";

describe("HealthController", () => {
  it("프로세스가 요청을 받을 수 있으면 ok를 반환한다", async () => {
    const moduleRef = await Test.createTestingModule({
      controllers: [HealthController],
    }).compile();

    const controller = moduleRef.get(HealthController);
    expect(controller.check()).toEqual({ status: "ok" });
  });
});
```

- [ ] **Step 3: 테스트가 실패하는지 확인**

Run: `npm test -- health.controller.spec.ts`

Expected: `HealthController`가 존재하지 않아 FAIL.

- [ ] **Step 4: 최소 HealthController 구현**

```typescript
import { Controller, Get } from "@nestjs/common";

@Controller("health")
export class HealthController {
  @Get()
  check(): { status: "ok" } {
    return { status: "ok" };
  }
}
```

`HealthController`를 `AppModule.controllers`에 등록한다. `main.ts`에는 `ValidationPipe`를 `whitelist: true`, `transform: true`로 등록한다.
Nest CLI가 만든 예제 `AppController`와 `AppService` 파일은 삭제하고 `AppModule`에서도 제거한다.

- [ ] **Step 5: 검사 실행**

Run: `npm test -- health.controller.spec.ts && npm run lint && npm run build`

Expected: 모든 명령 PASS.

- [ ] **Step 6: 커밋**

```bash
git add package.json package-lock.json nest-cli.json tsconfig*.json eslint.config.mjs src test
git commit -m "feat: bootstrap ticketing server"
```

---

### Task 2: PostgreSQL과 Prisma 연결

**Files:**

- Create: `docker-compose.yml`
- Create: `.env.example`
- Create: `prisma.config.ts`
- Create: `prisma/schema.prisma`
- Create: `src/database/database.module.ts`
- Create: `src/database/prisma.service.ts`
- Test: `test/database.integration-spec.ts`
- Modify: `src/app.module.ts`

**Interfaces:**

- Produces: 전역 주입 가능한 `PrismaService`
- Produces: 개발 DB `postgresql://ticketing:ticketing@localhost:5432/ticketing`

- [ ] **Step 1: 의존성과 PostgreSQL 컨테이너 준비**

Run:

```bash
npm install @prisma/client @prisma/adapter-pg pg dotenv
npm install -D prisma @types/pg
docker compose up -d postgres
```

`docker-compose.yml`의 `postgres` 서비스는 PostgreSQL 공식 이미지, database/user/password `ticketing`, host port `5432`, named volume `postgres-data`, healthcheck `pg_isready -U ticketing`을 사용한다.

- [ ] **Step 2: 연결 통합 테스트 작성**

```typescript
let prisma: PrismaService;

beforeAll(async () => {
  process.env.DATABASE_URL = process.env.DATABASE_URL_TEST;
  prisma = new PrismaService();
  await prisma.onModuleInit();
});

afterAll(async () => {
  await prisma.onModuleDestroy();
});

describe("Database connection", () => {
  it("PostgreSQL에 질의할 수 있다", async () => {
    const rows = await prisma.$queryRaw<Array<{ value: number }>>`
      SELECT 1 AS value
    `;
    expect(rows).toEqual([{ value: 1 }]);
  });
});
```

- [ ] **Step 3: DB 연결 코드가 없어 실패하는지 확인**

Run: `npm test -- database.integration-spec.ts --runInBand`

Expected: `PrismaService` 또는 generated client를 찾지 못해 FAIL.

- [ ] **Step 4: Prisma 7 설정과 PrismaService 구현**

`prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
}
```

`src/database/prisma.service.ts`에서 `PrismaPg`에 `DATABASE_URL`과 `max: 5`, `connectionTimeoutMillis: 5000`, `idleTimeoutMillis: 30000`을 전달한다. 생성한 adapter로 `PrismaClient`를 상속하고 Nest lifecycle에서 `$connect()`와 `$disconnect()`를 호출한다.

`DatabaseModule`은 `@Global()` 모듈로 선언하고 `PrismaService`를 export한다.
`.env.example`에는 개발용 `DATABASE_URL`과 별도 database 이름을 사용하는 `DATABASE_URL_TEST`를 모두 기록한다.

- [ ] **Step 5: Prisma client 생성과 연결 테스트**

Run:

```bash
npx prisma generate
npm test -- database.integration-spec.ts --runInBand
```

Expected: 통합 테스트 PASS. `pg_stat_activity`에서 테스트 프로세스의 연결을 확인할 수 있다.

- [ ] **Step 6: 커밋**

```bash
git add package.json package-lock.json docker-compose.yml .env.example prisma.config.ts prisma src/database src/app.module.ts test/database.integration-spec.ts
git commit -m "feat: connect server to postgresql with prisma"
```

---

### Task 3: 열차 운행과 좌석 조회

**Files:**

- Modify: `prisma/schema.prisma`
- Create: `prisma/seed.ts`
- Create: `src/train/train.module.ts`
- Create: `src/train/train.controller.ts`
- Create: `src/train/train.service.ts`
- Create: `src/train/train.repository.ts`
- Create: `src/train/dto/search-train-runs.query.ts`
- Test: `src/train/train.service.spec.ts`
- Test: `test/train-search.e2e-spec.ts`
- Modify: `src/app.module.ts`

**Interfaces:**

- Produces: `TrainService.search(departureStation, arrivalStation, serviceDate)`
- Produces: `GET /train-runs?departureStation=SEOUL&arrivalStation=BUSAN&serviceDate=YYYY-MM-DD`
- Produces: `GET /train-runs/:trainRunId/seats`로 좌석 ID, 객차, 좌석 번호, 가격과 상태 조회

- [ ] **Step 1: 검색 service 실패 테스트 작성**

```typescript
it("출발지, 도착지와 운행일이 일치하는 열차를 반환한다", async () => {
  repository.findRuns.mockResolvedValue([{ id: "run-1", trainNumber: "KTX-101", availableSeatCount: 20 }]);

  await expect(service.search("SEOUL", "BUSAN", "2026-08-01")).resolves.toEqual([{ id: "run-1", trainNumber: "KTX-101", availableSeatCount: 20 }]);
});
```

- [ ] **Step 2: 실패 확인**

Run: `npm test -- train.service.spec.ts`

Expected: `TrainService.search`가 없어 FAIL.

- [ ] **Step 3: 최소 검색 구현**

`TrainRunRepository`는 Prisma를 사용해 출발지, 도착지, 운행일로 `TrainRun`을 조회하고 `AVAILABLE` 상태 좌석 수를 함께 반환한다. `TrainService`는 빈 출발지·도착지와 잘못된 날짜를 거부한다. `listAvailableSeats(trainRunId)`는 해당 운행의 좌석 목록을 반환한다. Controller DTO는 `class-validator`로 입력을 검증한다.

- [ ] **Step 4: 스키마와 seed 추가**

`TrainRun`은 열차 번호, 역 코드, 출발·도착 시각을 가진다. `Seat`는 `trainRunId`, 객차 번호, 좌석 번호, 가격과 `AVAILABLE | HELD | RESERVED` 상태를 가진다. `(trainRunId, carNumber, seatNumber)`에 unique 제약을 둔다.

Run:

```bash
npx prisma migrate dev --name add_train_runs_and_seats
npx prisma db seed
```

Expected: 서울-부산 운행 1건과 좌석 20개가 생성된다.

- [ ] **Step 5: 단위·E2E 테스트 실행**

Run: `npm test -- train.service.spec.ts && npm run test:e2e -- train-search.e2e-spec.ts`

Expected: 정상 검색 PASS, 필수 query 누락은 HTTP 400.

- [ ] **Step 6: 커밋**

```bash
git add prisma src/train src/app.module.ts test/train-search.e2e-spec.ts
git commit -m "feat: add train run and seat search"
```

---

### Task 4: 좌석 임시 점유

**Files:**

- Modify: `prisma/schema.prisma`
- Create: `src/seat-hold/seat-hold.module.ts`
- Create: `src/seat-hold/seat-hold.controller.ts`
- Create: `src/seat-hold/seat-hold.service.ts`
- Create: `src/seat-hold/seat-hold.repository.ts`
- Create: `src/seat-hold/dto/create-seat-hold.dto.ts`
- Create: `src/common/errors/domain-error.ts`
- Create: `src/common/errors/domain-exception.filter.ts`
- Test: `src/seat-hold/seat-hold.service.spec.ts`
- Test: `test/seat-hold.e2e-spec.ts`
- Modify: `src/app.module.ts`

**Interfaces:**

- Produces: `SeatHoldService.create({ trainRunId, seatId, userId })`
- Produces: `POST /seat-holds -> { holdId, expiresAt }`

- [ ] **Step 1: 좌석 점유 규칙 테스트 작성**

```typescript
it("AVAILABLE 좌석을 5분간 HELD 상태로 만든다", async () => {
  repository.findSeat.mockResolvedValue({ id: "seat-1", status: "AVAILABLE" });
  repository.createHold.mockResolvedValue({
    id: "hold-1",
    expiresAt: new Date("2026-08-01T00:05:00.000Z"),
  });

  const result = await service.create({
    trainRunId: "run-1",
    seatId: "seat-1",
    userId: "user-1",
  });

  expect(result.id).toBe("hold-1");
  expect(repository.createHold).toHaveBeenCalledTimes(1);
});

it("이미 점유된 좌석은 SEAT_NOT_AVAILABLE 오류를 반환한다", async () => {
  repository.findSeat.mockResolvedValue({ id: "seat-1", status: "HELD" });
  await expect(
    service.create({
      trainRunId: "run-1",
      seatId: "seat-1",
      userId: "user-1",
    }),
  ).rejects.toMatchObject({
    code: "SEAT_NOT_AVAILABLE",
  });
});
```

- [ ] **Step 2: 실패 확인**

Run: `npm test -- seat-hold.service.spec.ts`

Expected: service와 오류 타입이 없어 FAIL.

- [ ] **Step 3: 점유 service와 repository 구현**

service는 좌석 상태를 확인하고 5분 만료 시각을 계산한다. repository는 Prisma transaction 안에서 `Seat.status`를 `HELD`로 변경하고 `SeatHold`를 생성한다. 이 단계에서는 명시적 row lock을 사용하지 않으며, 2단계 동시성 실험에서 현재 구현의 경쟁 조건을 재현한다.

- [ ] **Step 4: HTTP 오류 변환 구현**

`DomainError`는 `code`와 `message`를 가진다. 전역 exception filter는 `SEAT_NOT_AVAILABLE`을 HTTP 409로 변환하고 나머지 예상하지 못한 오류는 HTTP 500으로 처리한다.

- [ ] **Step 5: 테스트 실행**

Run: `npm test -- seat-hold.service.spec.ts && npm run test:e2e -- seat-hold.e2e-spec.ts`

Expected: 첫 점유 HTTP 201, 동일 좌석의 순차 재요청 HTTP 409.

- [ ] **Step 6: 커밋**

```bash
git add prisma src/seat-hold src/common src/app.module.ts test/seat-hold.e2e-spec.ts
git commit -m "feat: add temporary seat holds"
```

---

### Task 5: 가상 결제와 예매 확정

**Files:**

- Modify: `prisma/schema.prisma`
- Create: `src/payment/payment.port.ts`
- Create: `src/payment/fake-payment.adapter.ts`
- Create: `src/reservation/reservation.module.ts`
- Create: `src/reservation/reservation.controller.ts`
- Create: `src/reservation/reservation.service.ts`
- Create: `src/reservation/reservation.repository.ts`
- Create: `src/reservation/dto/create-reservation.dto.ts`
- Test: `src/reservation/reservation.service.spec.ts`
- Test: `test/reservation.e2e-spec.ts`
- Modify: `src/app.module.ts`

**Interfaces:**

- Consumes: 유효한 `SeatHold`
- Produces: `PaymentPort.pay({ paymentKey, amount }) -> { approved: boolean }`
- Produces: `POST /reservations -> { reservationId, status: "CONFIRMED" }`

- [ ] **Step 1: 예매 service 실패 테스트 작성**

```typescript
it("유효한 점유의 결제가 승인되면 예매를 확정한다", async () => {
  repository.findActiveHold.mockResolvedValue({
    id: "hold-1",
    userId: "user-1",
    seatId: "seat-1",
    price: 59000,
  });
  payment.pay.mockResolvedValue({ approved: true });
  repository.confirm.mockResolvedValue({ id: "reservation-1", status: "CONFIRMED" });

  await expect(
    service.create({
      holdId: "hold-1",
      userId: "user-1",
      paymentKey: "pay_1",
    }),
  ).resolves.toEqual({
    id: "reservation-1",
    status: "CONFIRMED",
  });
});

it("점유 소유자가 다르면 HOLD_OWNERSHIP_MISMATCH 오류를 반환한다", async () => {
  repository.findActiveHold.mockResolvedValue({
    id: "hold-1",
    userId: "user-2",
    seatId: "seat-1",
    price: 59000,
  });
  await expect(
    service.create({
      holdId: "hold-1",
      userId: "user-1",
      paymentKey: "pay_1",
    }),
  ).rejects.toMatchObject({
    code: "HOLD_OWNERSHIP_MISMATCH",
  });
});
```

- [ ] **Step 2: 실패 확인**

Run: `npm test -- reservation.service.spec.ts`

Expected: reservation과 payment port가 없어 FAIL.

- [ ] **Step 3: 가상 결제 port 구현**

`FakePaymentAdapter`는 `paymentKey`가 `fail_`로 시작하면 `{ approved: false }`, 그 외에는 `{ approved: true }`를 반환한다. 외부 대기 시간을 흉내 내기 위한 sleep은 넣지 않는다.

- [ ] **Step 4: 예매 확정 구현**

service는 점유 존재 여부, 소유자와 만료 시간을 확인한 뒤 결제를 호출한다. 결제가 승인된 경우 repository transaction으로 `Reservation`, `Payment`를 생성하고 좌석을 `RESERVED`, 점유를 `CONVERTED`로 변경한다. 결제가 거절되면 `PAYMENT_DECLINED`를 반환한다.

- [ ] **Step 5: 단위·E2E 테스트 실행**

Run: `npm test -- reservation.service.spec.ts && npm run test:e2e -- reservation.e2e-spec.ts`

Expected: 승인 결제 HTTP 201, 거절 결제 HTTP 402, 다른 사용자 점유 사용 HTTP 403.

- [ ] **Step 6: 커밋**

```bash
git add prisma src/payment src/reservation src/app.module.ts test/reservation.e2e-spec.ts
git commit -m "feat: confirm reservations with fake payments"
```

---

### Task 6: 예매 취소

**Files:**

- Modify: `src/reservation/reservation.controller.ts`
- Modify: `src/reservation/reservation.service.ts`
- Modify: `src/reservation/reservation.repository.ts`
- Test: `src/reservation/reservation.service.spec.ts`
- Test: `test/reservation-cancel.e2e-spec.ts`

**Interfaces:**

- Produces: `ReservationService.cancel(reservationId, userId)`
- Produces: `DELETE /reservations/:reservationId -> { status: "CANCELLED" }`

- [ ] **Step 1: 취소 규칙 테스트 작성**

```typescript
it("확정 예매를 취소하고 좌석을 다시 AVAILABLE로 만든다", async () => {
  repository.findById.mockResolvedValue({
    id: "reservation-1",
    userId: "user-1",
    status: "CONFIRMED",
    seatId: "seat-1",
  });
  repository.cancel.mockResolvedValue({ id: "reservation-1", status: "CANCELLED" });

  await expect(service.cancel("reservation-1", "user-1")).resolves.toEqual({
    id: "reservation-1",
    status: "CANCELLED",
  });
});
```

- [ ] **Step 2: 실패 확인**

Run: `npm test -- reservation.service.spec.ts`

Expected: `cancel`이 없어 FAIL.

- [ ] **Step 3: 취소 구현**

service는 예매 존재 여부, 소유자와 `CONFIRMED` 상태를 검증한다. repository transaction은 예매와 결제를 취소 상태로 바꾸고 좌석을 `AVAILABLE`로 되돌린다. 이미 취소된 예매의 재요청은 현재 `CANCELLED` 결과를 반환해 멱등하게 처리한다.

- [ ] **Step 4: 테스트 실행**

Run: `npm test -- reservation.service.spec.ts && npm run test:e2e -- reservation-cancel.e2e-spec.ts`

Expected: 첫 취소와 반복 취소 모두 성공하고 좌석은 `AVAILABLE`.

- [ ] **Step 5: 커밋**

```bash
git add src/reservation test/reservation-cancel.e2e-spec.ts
git commit -m "feat: cancel confirmed reservations"
```

---

### Task 7: 전체 예매 E2E와 테스트 격리

**Files:**

- Create: `test/setup.ts`
- Create: `test/booking-flow.e2e-spec.ts`
- Modify: `test/jest-e2e.json`
- Modify: `package.json`

**Interfaces:**

- Consumes: 열차 검색, 좌석 점유, 예매 확정, 예매 취소 API
- Produces: `npm run test:e2e`로 반복 실행 가능한 전체 흐름 검증

- [ ] **Step 1: 전체 흐름 테스트 작성**

```typescript
it("열차 조회부터 예매와 취소까지 완료한다", async () => {
  const serviceDate = "2026-08-01";
  const search = await request(app.getHttpServer()).get("/train-runs").query({ departureStation: "SEOUL", arrivalStation: "BUSAN", serviceDate }).expect(200);

  const seats = await request(app.getHttpServer()).get(`/train-runs/${search.body[0].id}/seats`).expect(200);

  const hold = await request(app.getHttpServer())
    .post("/seat-holds")
    .send({
      trainRunId: search.body[0].id,
      seatId: seats.body[0].id,
      userId: "user-1",
    })
    .expect(201);

  const reservation = await request(app.getHttpServer()).post("/reservations").send({ holdId: hold.body.holdId, userId: "user-1", paymentKey: "pay_1" }).expect(201);

  await request(app.getHttpServer()).delete(`/reservations/${reservation.body.reservationId}`).send({ userId: "user-1" }).expect(200);
});
```

- [ ] **Step 2: 테스트 데이터 초기화 구현**

`test/setup.ts`는 각 E2E suite 시작 전에 테스트 DB의 reservation, payment, seat hold, seat, train run 순서로 데이터를 삭제하고 고정된 운행과 좌석을 생성한다. 개발 DB와 다른 `DATABASE_URL_TEST`가 없으면 테스트를 즉시 실패시켜 개발 데이터를 삭제하지 않게 한다.

- [ ] **Step 3: 전체 검사 실행**

Run:

```bash
npm run lint
npm test -- --runInBand
npm run test:e2e -- --runInBand
npm run build
```

Expected: 모든 검사 PASS. 동일 명령을 두 번 실행해도 결과가 같다.

- [ ] **Step 4: 커밋**

```bash
git add test package.json package-lock.json
git commit -m "test: cover the basic booking journey"
```

---

### Task 8: Winston JSON 로그

**Files:**

- Create: `src/common/logging/logging.module.ts`
- Create: `src/common/logging/logging.interceptor.ts`
- Test: `src/common/logging/logging.interceptor.spec.ts`
- Modify: `src/main.ts`
- Modify: `src/app.module.ts`
- Modify: `package.json`

**Interfaces:**

- Consumes: HTTP request와 response
- Produces: stdout JSON `{ level, message, requestId, method, path, statusCode, durationMs, service }`

- [ ] **Step 1: 로그 interceptor 실패 테스트 작성**

```typescript
it("요청 완료 시 requestId와 처리 시간을 기록한다", async () => {
  await lastValueFrom(interceptor.intercept(context, next));
  expect(logger.info).toHaveBeenCalledWith(
    "http_request_completed",
    expect.objectContaining({
      requestId: "req-1",
      method: "GET",
      path: "/health",
      statusCode: 200,
    }),
  );
});
```

- [ ] **Step 2: 실패 확인**

Run: `npm test -- logging.interceptor.spec.ts`

Expected: logging module과 interceptor가 없어 FAIL.

- [ ] **Step 3: Winston stdout 설정**

Run: `npm install winston nest-winston`

Winston transport는 `Console` 하나만 사용하고 production format은 JSON으로 설정한다. 개발 환경도 동일한 필드 구조를 유지한다. `LoggingInterceptor`는 요청 헤더의 `x-request-id`를 사용하거나 UUID를 생성하고 응답 헤더에도 같은 값을 기록한다.

- [ ] **Step 4: 민감 정보 제외 확인**

로그 metadata에는 request body, authorization header, payment key를 넣지 않는다. 오류 로그에는 오류 이름, 메시지, stack, requestId만 포함한다.

- [ ] **Step 5: 전체 검사와 수동 확인**

Run:

```bash
npm test -- logging.interceptor.spec.ts
npm run build
npm run start:dev
```

다른 터미널에서 Run: `curl -i http://localhost:3000/health`

Expected: 응답의 `x-request-id`와 stdout JSON의 `requestId`가 같고 health 응답은 HTTP 200.

- [ ] **Step 6: 커밋**

```bash
git add package.json package-lock.json src/common/logging src/main.ts src/app.module.ts
git commit -m "feat: add structured winston request logs"
```

---

## Phase 1 Completion Gate

다음 명령이 모두 통과해야 2단계 DB 동시성 계획으로 이동한다.

```bash
npm run lint
npm test -- --runInBand
npm run test:e2e -- --runInBand
npm run build
docker compose ps
```

수동으로 다음 질문에 답할 수 있어야 한다.

1. NestJS module, controller, service와 repository의 책임은 무엇인가?
2. 단위 테스트에서 repository를 mock하고 E2E에서 실제 PostgreSQL을 사용하는 이유는 무엇인가?
3. 하나의 `server` 프로세스가 Prisma connection pool을 하나만 가져야 하는 이유는 무엇인가?
4. transaction 안에서 함께 성공하거나 함께 실패해야 하는 DB 변경은 무엇인가?
5. 현재 좌석 점유 구현이 동시 요청에서 안전하다고 아직 보장할 수 없는 이유는 무엇인가?

다섯 질문에 답하고 전체 검사가 통과하면 다음 계획에서 k6로 경쟁 조건을 재현하고 PostgreSQL row lock과 connection pool을 학습한다.
