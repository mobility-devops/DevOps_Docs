# DevOps Project Backend Architecture

> 원본: 노션 「DevOps Project Backend Architecture」에서 2026-09-30 기준으로 옮김.

## 택시 배차 서비스 백엔드 설계

> 📌 **이 문서는 최종 목표(전체) 설계.** 현재 구현된 첫 버전(v1)은 이 중 일부만 포함 (「아키텍처 개요」 1장 앱 트랙 기준).
>
> - **v1 포함:** 사용자·기사·차량·호출 CRUD, 호출 → 수락 → 도착 → 시작 → 완료/취소 흐름. DB는 MySQL 8 + Flyway.
> - **v1 제외 (이후 버전):** 로그인·JWT·Spring Security, Redis·위치·자동 배차, `dispatch_offers`, SSE, outbox, idempotency, `ride_events`.
> - **v1 사용자 식별:** 요청 헤더 `X-User-Id` (누락 401, 없는 사용자 404, 역할 불일치 403). `users`는 id, name, phone, role(`PASSENGER`/`DRIVER`).
> - **v1 수락 방식(A안):** 기사가 `GET /api/v1/rides?status=SEARCHING`으로 목록을 보고 `POST /api/v1/rides/{rideId}/accept`로 직접 수락. 중복 수락은 `rides.version`(낙관적 락)과 `active_assignments`로 방지.
> - **패키지 루트:** 실제 코드는 `kim.autoever.taxi`.

### 1. 구현 범위

#### 승객 기능

- 로그인
- **출발지·목적지를 입력하여 택시 호출**
- **배차 진행 상태 조회(배차 확정 여부 확인용)**
- 배정 기사와 차량 정보 확인
- 탑승 전 호출 취소
- **운행 결과 조회**

#### 기사 기능

- 로그인
- 온라인·오프라인 상태 변경
- 현재 위치 전송
- **배차 제안 조회·수락·거절**
- 출발지 도착 처리
- **운행 시작·완료 처리**

#### 서버 자동 처리

- **온라인 상태인 기사 중 랜덤 배정**
- 주변 기사 검색
- 기사에게 순차적으로 배차 제안
- 거절·만료 시 다음 기사 탐색
- 배차 탐색 시간 초과 처리
- 호출·운행 상태 변경 알림

#### 첫 버전에서 제외할 기능

- 실제 결제·정산
- 예약 호출
- 합승
- 복잡한 요금 정책
- 실제 도로 기반 최적 배차

---

### 2. 기술 구성

| 구분 | 기술 | 용도 |
| --- | --- | --- |
| 백엔드 | Java, Spring Boot | API와 업무 로직 |
| HTTP API | Spring Web | REST API |
| 인증·권한 | Spring Security, JWT | 승객·기사 인증 |
| DB 접근 | Spring Data JPA | 데이터 저장·조회 |
| 영구 저장소 | MySQL 8 (db-01 전용 VM) | 호출·배차·운행 기록 |
| 위치 저장소 | Redis | 최신 기사 위치·주변 기사 검색 |
| 실시간 알림 | SSE | 배차 제안·상태 변경 전달 |
| API 명세 | OpenAPI | 요청·응답 문서화 |
| DB 변경 관리 | Flyway | 테이블 변경 이력 관리 |

#### 애플리케이션 구성

하나의 Spring Boot 프로젝트에서 기능별 패키지를 분리한다.

```text
kim.autoever.taxi
├── auth
├── user
├── driver
├── location
├── ride
├── dispatch
├── notification
└── common
```

각 기능 안에서는 다음 구조를 사용한다.

```text
ride
├── controller
├── service
├── repository
├── domain
└── dto
```

| 계층 | 책임 |
| --- | --- |
| Controller | 요청 수신, 입력 검증, 응답 반환 |
| Service | 권한·상태 검사, 업무 처리, 트랜잭션 |
| Repository | DB 저장·조회·잠금 |
| Domain | 엔티티, 상태, 상태 전환 규칙 |
| DTO | API 요청·응답 형식 |

**Controller에서 배차 로직을 처리하지 않고, JPA 엔티티를 API 응답으로 직접 반환하지 않는다.**

---

### 3. 데이터 저장 설계

#### MySQL 8 테이블

| 테이블 | 주요 필드 | 역할 |
| --- | --- | --- |
| `users` | id, email, password_hash, role | 사용자·권한 |
| `drivers` | id, user_id, availability | 기사 정보·접속 상태 |
| `vehicles` | id, driver_id, plate_number, model | 차량 정보 |
| `rides` | id, passenger_id, driver_id, 출발지, 목적지, status, version | 호출부터 완료까지의 상태 |
| `dispatch_offers` | id, ride_id, driver_id, status, expires_at | 기사별 배차 제안 |
| `active_passenger_rides` | passenger_id, ride_id | 승객의 중복 활성 호출 방지 |
| `active_assignments` | driver_id, ride_id | 기사의 중복 활성 배정 방지 |
| `ride_events` | id, ride_id, 이전 상태, 변경 상태, actor_id, created_at | 상태 변경 이력 |
| `idempotency_requests` | 사용자, API, 요청 키, 요청 해시, 결과 | 중복 요청 처리 |
| `outbox_events` | id, event_type, payload, 처리 상태, 재시도 시각 | 전달할 이벤트·후속 작업 |

#### 주요 제약조건

- 사용자 이메일은 유일하다.
- 기사 정보는 기사 사용자 한 명당 하나다.
- 승객 한 명은 진행 중인 호출을 하나만 가질 수 있다.
- 기사 한 명은 활성 운행에 하나만 배정될 수 있다.
- 호출 하나에는 기사 한 명만 배정될 수 있다.

활성 호출·배정 테이블은 완료 또는 취소 시 해제한다. 기존 `rides` 기록은 유지한다.

#### Redis 저장 내용

```text
기사 위치 GEO 인덱스
기사별 최신 위치 수신 시각
기사별 위치 순서 번호
Pod 사이의 실시간 알림 채널
```

Redis의 기사 위치와 상태는 후보 탐색에 사용한다. **최종 배차 가능 여부는 MySQL에서 다시 확인한다.**

---

### 4. 상태 정의

#### 호출·운행 상태

| 상태 | 의미 |
| --- | --- |
| `SEARCHING` | 기사 탐색 중 |
| `ASSIGNED` | 기사 배정 완료 |
| `ARRIVED` | 기사 출발지 도착 |
| `IN_PROGRESS` | 승객 탑승·운행 중 |
| `COMPLETED` | 운행 완료 |
| `CANCELLED` | 호출 취소 |
| `NO_DRIVER` | 제한 시간 내 배차 실패 |

```text
SEARCHING → ASSIGNED → ARRIVED → IN_PROGRESS → COMPLETED

SEARCHING → CANCELLED / NO_DRIVER
ASSIGNED → CANCELLED
ARRIVED → CANCELLED
```

MVP에서는 승객의 일반 취소를 탑승 전까지 허용한다.

#### 배차 제안 상태

```text
PENDING → ACCEPTED
        → REJECTED
        → EXPIRED
        → REVOKED
```

호출 취소 등으로 유효하지 않게 된 제안은 `REVOKED`로 처리한다.

#### 기사 상태

기사의 온라인 여부와 배차 여부를 분리한다.

- `availability`: `ONLINE` / `OFFLINE`
- 활성 운행 여부: `active_assignments`로 판단

기사 앱이 임의로 자신을 “배차 가능”으로 바꾸어 기존 운행을 해제할 수 없도록 한다.

---

### 5. API 공통 규칙

#### 기본 경로와 인증

```text
/api/v1
```

```text
Authorization: Bearer <accessToken>
```

사용자 ID는 인증 정보에서 가져온다. 요청 본문에 있는 사용자 ID를 신뢰하지 않는다.

#### 요청 형식

- JSON 사용
- 시각은 UTC 기준 ISO 8601 사용
- 좌표 범위 검증
- 오류 응답에 추적용 `requestId` 포함

#### 오류 응답 예시

```text
{
  "code": "OFFER_EXPIRED",
  "message": "배차 제안이 만료되었습니다.",
  "requestId": "req_123"
}
```

| HTTP 상태 | 용도 |
| --- | --- |
| `400` | 입력값 오류 |
| `401` | 인증 실패 |
| `403` | 권한 없음 |
| `404` | 리소스 없음 |
| `409` | 상태 충돌·중복 활성 호출 |
| `429` | 요청 제한 초과 |
| `503` | 일시적으로 처리 불가 |

---

### 6. API 목록

#### 인증·사용자

| 메서드 | 경로 | 기능 |
| --- | --- | --- |
| POST | `/auth/signup` | 승객 회원가입 |
| POST | `/auth/login` | 로그인 |
| GET | `/me` | 내 정보 조회 |

#### 기사

| 메서드 | 경로 | 기능 |
| --- | --- | --- |
| PUT | `/drivers/me/availability` | 온라인·오프라인 변경 |
| PUT | `/drivers/me/location` | 위치 갱신 |
| GET | `/drivers/me/offers` | 유효한 대기 제안 조회 |

#### 호출·운행

| 메서드 | 경로 | 권한·기능 |
| --- | --- | --- |
| POST | `/rides` | 승객 호출 생성 |
| GET | `/rides/current` | 본인의 활성 호출·운행 조회 |
| GET | `/rides/{rideId}` | 해당 승객·배정 기사 상세 조회 |
| POST | `/rides/{rideId}/cancel` | 해당 승객의 탑승 전 취소 |
| POST | `/rides/{rideId}/arrive` | 배정 기사의 도착 처리 |
| POST | `/rides/{rideId}/start` | 배정 기사의 운행 시작 |
| POST | `/rides/{rideId}/complete` | 배정 기사의 운행 완료 |

`GET /rides/current`는 활성 운행이 없으면 `200`과 `{"ride": null}`을 반환한다.

#### 배차·알림

| 메서드 | 경로 | 기능 |
| --- | --- | --- |
| POST | `/dispatch-offers/{offerId}/accept` | 제안 수락 |
| POST | `/dispatch-offers/{offerId}/reject` | 제안 거절 |
| GET | `/events` | 본인 이벤트 SSE 구독 |

---

### 7. 핵심 API 상세

#### 7.1 호출 생성

```text
POST /api/v1/rides
Idempotency-Key: <UUID>
```

```text
{
  "pickup": {
    "latitude": 37.4979,
    "longitude": 127.0276
  },
  "destination": {
    "latitude": 37.5665,
    "longitude": 126.9780
  }
}
```

**처리 순서**

1. 승객 권한과 좌표를 검증한다.
2. 같은 요청 키로 처리한 결과가 있는지 확인한다.
3. 진행 중인 호출이 있는지 확인한다.
4. `SEARCHING` 호출과 활성 호출 정보를 저장한다.
5. 배차 시작 작업과 중복 요청 결과를 같은 트랜잭션으로 저장한다.
6. 저장 완료 후 응답한다.

**응답: `201 Created`**

```text
{
  "rideId": "ride_123",
  "status": "SEARCHING",
  "version": 1
}
```

```text
Location: /api/v1/rides/ride_123
```

호출 생성 응답은 호출 접수 완료를 뜻한다. 기사 배정은 이후에 진행한다.

#### 7.2 기사 위치 갱신

```text
PUT /api/v1/drivers/me/location
```

```text
{
  "latitude": 37.4985,
  "longitude": 127.0280,
  "recordedAt": "2026-09-28T03:00:00Z",
  "sequence": 105
}
```

**처리 규칙**

- 인증된 기사 본인의 위치만 갱신한다.
- 과거 순서 번호가 늦게 도착하면 최신 위치를 덮어쓰지 않는다.
- 서버 수신 시각을 별도로 기록한다.
- 일정 시간 갱신이 없는 기사는 배차 후보에서 제외한다.

온라인 기사 위치 전송 간격은 초기값으로 3~5초를 사용하고, 테스트 후 조정한다.

#### 7.3 배차 제안 수락

```text
POST /api/v1/dispatch-offers/offer_123/accept
Idempotency-Key: <UUID>
```

**처리 순서**

하나의 DB 트랜잭션 안에서 다음을 수행한다.

1. 관련 호출·기사·제안을 정해진 순서로 잠근다.
2. 본인에게 전달된 제안인지 확인한다.
3. 제안 상태와 만료 시각을 확인한다.
4. 호출이 `SEARCHING`인지 확인한다.
5. 기사에게 다른 활성 운행이 없는지 확인한다.
6. 활성 배정 정보를 등록한다.
7. 호출을 `ASSIGNED`, 제안을 `ACCEPTED`로 변경한다.
8. 상태 이력과 알림 이벤트를 저장한다.
9. 커밋 후 결과를 반환한다.

**응답: `200 OK`**

```text
{
  "rideId": "ride_123",
  "status": "ASSIGNED",
  "version": 2
}
```

만료·취소·다른 활성 배정 등으로 수락할 수 없으면 `409`를 반환한다. 같은 수락의 재시도는 기존 성공 결과를 반환할 수 있도록 한다.

#### 7.4 운행 상태 변경

| API | 허용되는 이전 상태 | 변경 상태 |
| --- | --- | --- |
| `/arrive` | `ASSIGNED` | `ARRIVED` |
| `/start` | `ARRIVED` | `IN_PROGRESS` |
| `/complete` | `IN_PROGRESS` | `COMPLETED` |
| `/cancel` | `SEARCHING`, `ASSIGNED`, `ARRIVED` | `CANCELLED` |

상태 변경 시에는 권한 검사, 상태 검사, 변경 이력 저장, 알림 작업 저장을 수행한다.

완료·취소 시 활성 호출·배정 정보도 같은 트랜잭션으로 해제한다.

---

### 8. 배차 로직 구현

#### 첫 버전의 배차 정책

- 가까운 기사부터 순차적으로 제안한다.
- 거절하거나 만료되면 다음 후보로 이동한다.
- 이미 거절한 기사에게 같은 호출을 반복 제안하지 않는다.
- 전체 탐색 시간이 초과되면 `NO_DRIVER`로 종료한다.

| 설정 | 초기 실험값 |
| --- | --- |
| 검색 반경 | 1km → 3km → 5km |
| 기사 응답 제한 | 10초 |
| 전체 배차 탐색 제한 | 60초 |

이 값은 고정된 요구사항이 아니라 조정 가능한 설정값이다.

#### 주요 서비스 책임

| 서비스 | 책임 |
| --- | --- |
| `RideService` | 호출 생성·취소·운행 상태 변경 |
| `DriverLocationService` | 위치 저장·후보 조회 |
| `DispatchService` | 후보 선정·제안 생성 |
| `DispatchOfferService` | 수락·거절 처리 |
| `DispatchWorker` | 탐색 작업·제안 만료·재탐색 |
| `NotificationService` | 인증된 사용자에게 이벤트 전달 |

기사 응답을 기다리는 동안 HTTP 요청이나 DB 트랜잭션을 유지하지 않는다.

Worker 작업은 DB에 저장하고 점유 기한과 점유 버전을 검사한다. 여러 Pod가 실행되더라도 같은 작업의 결과가 중복 확정되지 않게 한다.

---

### 9. 중복 요청·실시간 알림

#### 중복 요청 방지

호출 생성 등 중요한 요청에는 `Idempotency-Key`를 사용한다.

- 사용자·API·키 조합에 UNIQUE 제약을 둔다.
- 같은 키와 같은 본문이면 기존 결과를 반환한다.
- 같은 키와 다른 본문이면 `409`를 반환한다.
- 업무 데이터와 처리 결과를 같은 트랜잭션에 저장한다.

#### 이벤트 저장

업무 상태와 전달할 이벤트를 함께 DB에 저장한다.

```text
배차 확정
→ rides 변경 + outbox_events 저장
→ COMMIT
→ Worker가 알림 전달
```

#### SSE 이벤트 예시

```text
id: evt_123
event: ride.assigned
data: {"rideId":"ride_123","status":"ASSIGNED","version":2}
```

대표 이벤트:

```text
dispatch.offer.created
dispatch.offer.expired
ride.assigned
ride.arrived
ride.started
ride.completed
ride.cancelled
ride.no_driver
```

화면은 이벤트 ID로 중복을 구분하고, 상태 버전으로 오래된 변경을 걸러낸다. 연결이 복구되면 현재 호출과 유효한 제안을 API로 다시 조회한다.

---

### 10. 구현 순서와 완료 기준

| 단계 | 구현 내용 | 완료 기준 |
| --- | --- | --- |
| 1 | 프로젝트·DB·인증 | 승객·기사 로그인과 권한 구분 |
| 2 | 호출·운행 API | 호출부터 완료까지 상태 전환 |
| 3 | 수락·취소 동시성 | 중복 활성 배정 방지 |
| 4 | 위치·자동 배차 | 주변 기사 순차 제안·만료 |
| 5 | SSE·Outbox | 상태 알림과 조회 복구 |
| 6 | Worker 재개·통합 검증 | 중단된 작업을 이어서 처리 |
