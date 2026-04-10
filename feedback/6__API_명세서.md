# 📡 API 명세서 — TicketFlow

> **문서 버전:** v1.0
> **최종 수정일:** 2026-04-09
> **연결 문서:** 3. 유스케이스 명세서, 4. 기능 명세서, 5. ERD

---

## 문서 읽는 법

| 아이콘 | 의미 |
|--------|------|
| 🔐 | JWT 인증 필수 (`Authorization: Bearer {token}`) |
| 🔐👑 | JWT 인증 + ADMIN 권한 필수 |
| ⚠️ | 동시성 민감 API — 분산락 또는 낙관적 락 적용 |
| 💾 | 캐싱 대상 API |
| 🔓 | 인증 불필요 (공개 API) |

**Base URL:** `http://api.ticketflow.io`

**공통 에러 응답 형식:**
```json
{
  "status": 409,
  "code": "SEAT_ALREADY_HELD",
  "message": "이미 선점된 좌석입니다.",
  "timestamp": "2026-04-09T12:00:00"
}
```

**WebSocket:** `ws://api.ticketflow.io/ws-stomp` (STOMP 프로토콜)

---

## 1. 인증 (Auth)

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| POST | `/api/auth/signup` | 회원가입 | `{ email, password, nickname }` | `201 { userId, email }` | 🔓 | UC-001 |
| POST | `/api/auth/login` | 로그인 (JWT 발급) | `{ email, password }` | `200 { accessToken }` | 🔓 | UC-002 |

### POST /api/auth/signup

**Request**
```json
{
  "email": "parksj@example.com",
  "password": "pass1234",
  "nickname": "티켓전사서준"
}
```

**Response 201 Created**
```json
{
  "userId": 1,
  "email": "parksj@example.com",
  "nickname": "티켓전사서준",
  "createdAt": "2026-04-09T10:00:00"
}
```

**Error Cases**
```json
// 409 — 이메일 중복
{ "status": 409, "code": "EMAIL_DUPLICATED", "message": "이미 가입된 이메일입니다." }

// 400 — 유효성 실패
{ "status": 400, "code": "VALIDATION_FAILED", "message": "비밀번호는 8자 이상, 영문+숫자를 포함해야 합니다." }
```

---

### POST /api/auth/login

**Request**
```json
{
  "email": "parksj@example.com",
  "password": "pass1234"
}
```

**Response 200 OK**
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjEsInJvbGUiOiJVU0VSIn0.abc123",
  "expiresIn": 3600,
  "tokenType": "Bearer"
}
```

**Error Cases**
```json
// 401 — 인증 실패
{ "status": 401, "code": "AUTHENTICATION_FAILED", "message": "이메일 또는 비밀번호가 올바르지 않습니다." }
```

---

## 2. 사용자 (Users)

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| GET | `/api/users/me` | 내 정보 조회 | — | `200 { userId, email, nickname, role, createdAt }` | 🔐 | UC-003 |
| PATCH | `/api/users/me` | 내 정보 수정 | `{ nickname?, password?, currentPassword }` | `200` | 🔐 | UC-003 |
| GET | `/api/users/me/bookings` | 내 예매(주문) 내역 조회 | `?page&size&status` | `200 Page<OrderSummary>` | 🔐 | UC-010 |
| GET | `/api/users/me/coupons` | 내 쿠폰 목록 조회 | — | `200 List<UserCoupon>` | 🔐 | UC-011 |

### GET /api/users/me

**Response 200 OK**
```json
{
  "userId": 1,
  "email": "parksj@example.com",
  "nickname": "티켓전사서준",
  "role": "USER",
  "createdAt": "2026-04-09T10:00:00"
}
```

---

### PATCH /api/users/me

**Request**
```json
{
  "nickname": "새닉네임",
  "currentPassword": "pass1234",
  "password": "newpass5678"
}
```

**Response 200 OK**
```json
{
  "message": "정보가 수정되었습니다."
}
```

**Error Cases**
```json
// 401 — 현재 비밀번호 불일치
{ "status": 401, "code": "INVALID_CURRENT_PASSWORD", "message": "현재 비밀번호가 올바르지 않습니다." }

// 409 — 닉네임 중복
{ "status": 409, "code": "NICKNAME_DUPLICATED", "message": "이미 사용 중인 닉네임입니다." }
```

---

### GET /api/users/me/bookings

**Query Parameters:** `?page=0&size=10&status=CONFIRMED`

**Response 200 OK**
```json
{
  "content": [
    {
      "orderId": 101,
      "status": "CONFIRMED",
      "eventTitle": "세븐틴 콘서트 2026 SPILL THE FEELS",
      "eventDate": "2026-05-10T18:00:00",
      "venueName": "KSPO돔",
      "totalAmount": 330000,
      "discountAmount": 5000,
      "finalAmount": 325000,
      "items": [
        { "seatNumber": "A구역-015", "sectionName": "A구역", "originalPrice": 110000 },
        { "seatNumber": "A구역-016", "sectionName": "A구역", "originalPrice": 110000 },
        { "seatNumber": "A구역-017", "sectionName": "A구역", "originalPrice": 110000 }
      ],
      "createdAt": "2026-04-09T20:05:00"
    }
  ],
  "totalPages": 1,
  "totalElements": 1,
  "page": 0,
  "size": 10
}
```

---

### GET /api/users/me/coupons

**Response 200 OK**
```json
[
  {
    "userCouponId": 10,
    "couponId": 3,
    "couponName": "신규 가입 5,000원 할인",
    "discountAmount": 5000,
    "expiredAt": "2026-05-31T23:59:59",
    "status": "ISSUED",
    "issuedAt": "2026-04-09T12:00:01"
  }
]
```

---

## 3. 이벤트 (Events)

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| POST | `/api/admin/events` | 이벤트 등록 | `{ title, category, venueId, eventDate, sections[] }` | `201 { eventId, title }` | 🔐👑 | UC-004 |
| GET | `/api/events` | 이벤트 목록 조회 💾 | `?page&size&sort&category` | `200 Page<EventSummary>` | 🔓 | UC-004/005 |
| GET | `/api/events/{eventId}` | 이벤트 상세 조회 💾 | — | `200 EventDetail` | 🔓 | UC-005 |

### POST /api/admin/events

**Request**
```json
{
  "title": "세븐틴 콘서트 2026 SPILL THE FEELS",
  "category": "CONCERT",
  "venueId": 1,
  "eventDate": "2026-05-10T18:00:00",
  "description": "세븐틴의 전국 투어 마지막 서울 공연",
  "thumbnailUrl": "https://cdn.ticketflow.io/events/seventeen-2026.jpg",
  "sections": [
    { "sectionName": "A구역", "price": 110000, "totalSeats": 200 },
    { "sectionName": "B구역", "price": 88000, "totalSeats": 350 },
    { "sectionName": "스탠딩", "price": 132000, "totalSeats": 150 }
  ]
}
```

**Response 201 Created**
```json
{
  "eventId": 42,
  "title": "세븐틴 콘서트 2026 SPILL THE FEELS",
  "totalSeats": 700,
  "sectionsCreated": 3
}
```

**Error Cases**
```json
// 403 — 권한 없음
{ "status": 403, "code": "FORBIDDEN", "message": "관리자 권한이 필요합니다." }

// 400 — 과거 일시
{ "status": 400, "code": "INVALID_EVENT_DATE", "message": "이벤트 일시는 현재 이후여야 합니다." }
```

---

### GET /api/events

**Query Parameters:** `?page=0&size=20&sort=eventDate,asc&category=CONCERT`

> 💾 Caffeine TTL 10분 (이벤트 등록 시 `@CacheEvict`)

**Response 200 OK**
```json
{
  "content": [
    {
      "eventId": 42,
      "title": "세븐틴 콘서트 2026 SPILL THE FEELS",
      "category": "CONCERT",
      "venueName": "KSPO돔",
      "eventDate": "2026-05-10T18:00:00",
      "minPrice": 88000,
      "remainingSeats": 523,
      "thumbnailUrl": "https://cdn.ticketflow.io/events/seventeen-2026.jpg"
    }
  ],
  "totalPages": 5,
  "totalElements": 98,
  "page": 0,
  "size": 20
}
```

---

### GET /api/events/{eventId}

> 💾 이벤트 기본 정보 Caffeine TTL 10분 / 잔여 좌석 수는 캐시 제외 (실시간)

**Response 200 OK**
```json
{
  "eventId": 42,
  "title": "세븐틴 콘서트 2026 SPILL THE FEELS",
  "category": "CONCERT",
  "venue": {
    "venueId": 1,
    "name": "KSPO돔",
    "address": "서울특별시 송파구 올림픽로 424"
  },
  "eventDate": "2026-05-10T18:00:00",
  "description": "세븐틴의 전국 투어 마지막 서울 공연",
  "thumbnailUrl": "https://cdn.ticketflow.io/events/seventeen-2026.jpg",
  "sections": [
    {
      "sectionId": 10,
      "sectionName": "A구역",
      "price": 110000,
      "totalSeats": 200,
      "remainingSeats": 87
    },
    {
      "sectionId": 11,
      "sectionName": "B구역",
      "price": 88000,
      "totalSeats": 350,
      "remainingSeats": 301
    },
    {
      "sectionId": 12,
      "sectionName": "스탠딩",
      "price": 132000,
      "totalSeats": 150,
      "remainingSeats": 135
    }
  ]
}
```

**Error Cases**
```json
// 404 — 이벤트 없음
{ "status": 404, "code": "EVENT_NOT_FOUND", "message": "존재하지 않는 이벤트입니다." }
```

---

## 4. 검색 (Search)

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| GET | `/api/v1/events/search` | 이벤트 검색 v1 (캐시 없음, 성능 기준선) | `?keyword&category&startDate&endDate&minPrice&maxPrice&page&size&sort` | `200 Page<EventSummary>` | 🔓 | UC-005 |
| GET | `/api/v2/events/search` | 이벤트 검색 v2 💾 (Caffeine → Redis Cache-Aside) | 동일 | `200 Page<EventSummary>` + `X-Cache: HIT/MISS` | 🔓 | UC-005 |
| GET | `/api/search/popular` | 인기 검색어 Top 10 💾 | — | `200 List<PopularKeyword>` | 🔓 | UC-006 |
| POST | `/api/search/popular/click` | 인기 검색어 클릭 점수 반영 (ZINCRBY) | `{ keyword }` | `200` | 🔓 | UC-006 |

### GET /api/v1/events/search

**Query Parameters:** `?keyword=뮤지컬&category=MUSICAL&startDate=2026-04-01&endDate=2026-04-30&minPrice=0&maxPrice=50000&page=0&size=20&sort=eventDate,asc`

> 캐시 미적용. 매 요청마다 MySQL 직접 조회. 응답 시간 로그 필수 기록 (v2 성능 비교용).

**Response 200 OK**
```json
{
  "content": [
    {
      "eventId": 15,
      "title": "오페라의 유령 2026 내한공연",
      "category": "MUSICAL",
      "venueName": "블루스퀘어 신한카드홀",
      "eventDate": "2026-04-20T19:30:00",
      "minPrice": 44000,
      "remainingSeats": 112,
      "thumbnailUrl": "https://cdn.ticketflow.io/events/phantom-2026.jpg"
    }
  ],
  "totalPages": 2,
  "totalElements": 31,
  "page": 0,
  "size": 20,
  "message": null
}
```

---

### GET /api/v2/events/search

> 💾 **Caffeine 캐시** (필수): `event-search` 캐시명, 캐시 키 = `{keyword}:{category}:{startDate}:{endDate}:{minPrice}:{maxPrice}:{page}:{size}:{sort}`, TTL 5분
>
> 💾 **Redis Cache-Aside** (도전): 동일 엔드포인트, 캐시 저장소만 전환
>
> 응답 헤더 `X-Cache: HIT` 또는 `X-Cache: MISS` 포함

**Response 200 OK** (캐시 히트 예시)
```http
HTTP/1.1 200 OK
X-Cache: HIT
Content-Type: application/json
```
```json
{
  "content": [
    {
      "eventId": 15,
      "title": "오페라의 유령 2026 내한공연",
      "category": "MUSICAL",
      "venueName": "블루스퀘어 신한카드홀",
      "eventDate": "2026-04-20T19:30:00",
      "minPrice": 44000,
      "remainingSeats": 112,
      "thumbnailUrl": "https://cdn.ticketflow.io/events/phantom-2026.jpg"
    }
  ],
  "totalPages": 2,
  "totalElements": 31,
  "page": 0,
  "size": 20
}
```

---

### GET /api/search/popular

> 💾 Redis ZSet (`ZREVRANGE search-keywords 0 9 WITHSCORES`)
> Redis 장애 시 빈 배열 반환 (Graceful Degradation, 검색 기능 영향 없음)

**Response 200 OK**
```json
[
  { "rank": 1, "keyword": "세븐틴", "count": 1420 },
  { "rank": 2, "keyword": "오페라의 유령", "count": 987 },
  { "rank": 3, "keyword": "KBO 올스타전", "count": 764 },
  { "rank": 4, "keyword": "뮤지컬", "count": 612 },
  { "rank": 5, "keyword": "아이유 콘서트", "count": 589 },
  { "rank": 6, "keyword": "연극", "count": 431 },
  { "rank": 7, "keyword": "스포츠", "count": 390 },
  { "rank": 8, "keyword": "발레", "count": 267 },
  { "rank": 9, "keyword": "클래식", "count": 214 },
  { "rank": 10, "keyword": "BTS", "count": 198 }
]
```

---

### POST /api/search/popular/click

> 인기 검색어 목록에서 키워드를 클릭할 때 서버 사이드에서 Redis ZSet 점수를 증가시킨다.
> `ZINCRBY search-keywords 1 {keyword}` 호출
> 검색창에 직접 입력하는 경우는 검색 API 호출 시 서버에서 자동으로 ZINCRBY를 처리하므로 이 API를 별도 호출할 필요 없음.
> **이 API는 "인기 검색어 클릭" 이벤트 전용**이다.

**Request**
```json
{ "keyword": "세븐틴" }
```

**Response 200 OK**
```json
{ "message": "검색어 점수가 반영되었습니다.", "keyword": "세븐틴" }
```

---

## 5. 좌석 & Hold (Seats)

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| GET | `/api/events/{eventId}/seats` | 구역별 좌석 상태 조회 | `?sectionId` (선택) | `200 SeatMap` | 🔐 | UC-007 |
| POST | `/api/events/{eventId}/seats/{seatId}/hold` | 좌석 임시 점유 (Hold) ⚠️ | — | `200 { holdToken, expiresAt }` | 🔐 | UC-007 |
| DELETE | `/api/events/{eventId}/seats/{seatId}/hold` | Hold 수동 해제 | — | `200` | 🔐 | UC-007 |

### GET /api/events/{eventId}/seats

> 좌석 상태 판단 로직 (ERD v7.0 기준):
> 1. `ACTIVE_BOOKING` 테이블에 해당 `seat_id` 존재 → **CONFIRMED**
> 2. Redis `hold:{eventId}:{seatId}` 키 존재 → **ON_HOLD**
> 3. 그 외 → **AVAILABLE**
> 좌석 수가 많으면 `?sectionId` 파라미터로 구역 단위 분리 조회 가능 (예: `?sectionId=10`)

**Response 200 OK**
```json
{
  "eventId": 42,
  "sections": [
    {
      "sectionId": 10,
      "sectionName": "A구역",
      "price": 110000,
      "seats": [
        { "seatId": 1001, "seatNumber": "A구역-001", "row": "A", "col": 1, "status": "AVAILABLE" },
        { "seatId": 1002, "seatNumber": "A구역-002", "row": "A", "col": 2, "status": "ON_HOLD" },
        { "seatId": 1003, "seatNumber": "A구역-003", "row": "A", "col": 3, "status": "CONFIRMED" }
      ]
    }
  ]
}
```

---

### POST /api/events/{eventId}/seats/{seatId}/hold ⚠️

> **동시성 제어:** Redis 분산락 (`lock:seat:{eventId}:{seatId}`, TTL 3초, Lua Script 원자적 해제)
> **Fail Fast 전략:** 락 획득 실패 시 즉시 409 반환 (재시도 없음 — 티켓팅 UX상 즉각 피드백 필수)
> **어뷰징 방지:** 사용자당 동시 Hold 최대 4석 (`user-holds:{userId}` SCARD)

**Response 200 OK**
```json
{
  "holdToken": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "seatId": 1001,
  "seatNumber": "A구역-001",
  "expiresAt": "2026-04-09T20:10:00"
}
```

**Error Cases**
```json
// 400 — Hold 수 초과 (어뷰징 방지)
{ "status": 400, "code": "HOLD_LIMIT_EXCEEDED", "message": "최대 4석까지만 선택할 수 있습니다." }

// 409 — 분산락 획득 실패 (Fail Fast)
{ "status": 409, "code": "SEAT_LOCK_FAILED", "message": "다른 사용자가 처리 중입니다. 다른 좌석을 선택해주세요." }

// 409 — 이미 Hold됨
{ "status": 409, "code": "SEAT_ALREADY_HELD", "message": "이미 선점된 좌석입니다." }

// 409 — 이미 예매됨
{ "status": 409, "code": "SEAT_ALREADY_CONFIRMED", "message": "이미 예매된 좌석입니다." }
```

---

### DELETE /api/events/{eventId}/seats/{seatId}/hold

**Response 200 OK**
```json
{
  "message": "좌석 선택이 해제되었습니다.",
  "seatId": 1001
}
```

**Error Cases**
```json
// 403 — 본인 Hold가 아님
{ "status": 403, "code": "HOLD_NOT_OWNED", "message": "본인이 선점한 좌석만 해제할 수 있습니다." }

// 404 — Hold 없음 (TTL 만료)
{ "status": 404, "code": "HOLD_NOT_FOUND", "message": "점유 정보가 없거나 이미 만료되었습니다." }
```

---

## 6. 예매 / 주문 (Orders)

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| POST | `/api/orders` | 주문 생성 (티켓 예매 요청) ⚠️ | `{ holdTokens[], couponId? }` | `200 { orderId, status, paymentAmount }` | 🔐 | UC-008 |
| POST | `/api/mock-pg/webhook` | Mock PG 웹훅 수신 (주문 확정) ⚠️ | `{ orderId, paymentStatus, paymentKey, paidAmount }` | `200` | 🔓 | UC-008 |
| POST | `/api/orders/{orderId}/cancel` | 주문 취소 | — | `200` | 🔐 | UC-009 |

### POST /api/orders ⚠️

> **동시성 제어 (쿠폰 적용 시):** `lock:user-coupon-use:{userCouponId}` 분산락 획득
> **락 순서:** ① 좌석 Hold 검증(Redis) → ② 쿠폰 락(Redis) → ③ DB 트랜잭션 (데드락 방지)
> **Hold TTL 연장 (L-02 대응):** 주문 생성 성공 시 모든 Hold 키의 TTL을 +5분 연장한다.
> 이로써 "주문 생성 → PG 웹훅 도착" 사이의 타이밍 갭 문제를 해결한다.
> (Redis `EXPIRE hold:{eventId}:{seatId} 300` 재실행)
> Hold 토큰 최대 4개, 단일 ORDER로 묶어 처리

**Request**
```json
{
  "holdTokens": [
    "f47ac10b-58cc-4372-a567-0e02b2c3d479",
    "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "11223344-5566-7788-99aa-bbccddeeff00"
  ],
  "couponId": 10
}
```

**Response 200 OK**
```json
{
  "orderId": 101,
  "status": "PENDING",
  "totalAmount": 330000,
  "discountAmount": 5000,
  "finalAmount": 325000,
  "items": [
    { "seatId": 1001, "seatNumber": "A구역-015", "originalPrice": 110000 },
    { "seatId": 1002, "seatNumber": "A구역-016", "originalPrice": 110000 },
    { "seatId": 1003, "seatNumber": "A구역-017", "originalPrice": 110000 }
  ],
  "pgRequestId": "PG-20260409-101"
}
```

**Error Cases**
```json
// 409 — Hold 만료
{ "status": 409, "code": "HOLD_EXPIRED", "message": "점유 시간이 만료된 좌석이 있습니다. 좌석을 다시 선택해주세요." }

// 400 — 쿠폰 무효
{ "status": 400, "code": "COUPON_INVALID", "message": "사용할 수 없는 쿠폰입니다." }

// 400 — 본인 쿠폰 아님
{ "status": 400, "code": "COUPON_NOT_OWNED", "message": "본인 소유의 쿠폰만 사용할 수 있습니다." }
```

---

### POST /api/mock-pg/webhook ⚠️

> **동시성 제어:** 쿠폰 사용 시 `lock:user-coupon-use:{userCouponId}` 분산락 + `@Version` 낙관적 락 (`USER_COUPON` 엔티티)
> **멱등성:** `orderId` 기준으로 중복 웹훅 처리 방지 (이미 CONFIRMED면 200 즉시 반환)
> **성공 시 원자적 처리:** ORDER/BOOKING → CONFIRMED, BOOKING.ticket_code 생성, ACTIVE_BOOKING INSERT, Hold 삭제 (단일 트랜잭션)

**Request** (Mock PG → 서버)
```json
{
  "orderId": 101,
  "paymentStatus": "SUCCESS",
  "paymentKey": "PG-KEY-2026040900001",
  "paidAmount": 325000
}
```

**Response 200 OK** (성공)
```json
{
  "message": "주문이 확정되었습니다.",
  "orderId": 101,
  "bookings": [
    { "bookingId": 201, "ticketCode": "TF-2026-A015-XYZ1", "seatInfo": "A구역 A열 15번" },
    { "bookingId": 202, "ticketCode": "TF-2026-A016-XYZ2", "seatInfo": "A구역 A열 16번" },
    { "bookingId": 203, "ticketCode": "TF-2026-A017-XYZ3", "seatInfo": "A구역 A열 17번" }
  ]
}
```

**Request** (결제 실패)
```json
{
  "orderId": 101,
  "paymentStatus": "FAIL",
  "paymentKey": null,
  "paidAmount": 0
}
```

**Response 200 OK** (실패 처리 — 웹훅은 항상 200 반환, 처리 결과는 내부 로그)
```json
{
  "message": "결제 실패로 주문이 취소되었습니다.",
  "orderId": 101
}
```

---

### POST /api/orders/{orderId}/cancel

> 이벤트 시작 24시간 전까지 취소 가능
> **동시성 제어 (L-04 대응):** `SELECT ... FOR UPDATE`로 BOOKING, ORDER, ACTIVE_BOOKING 행을 잠금
> 취소 트랜잭션 진행 중 다른 사용자의 새 예매 확정(웹훅)과의 충돌을 방지한다.
> **단일 트랜잭션 내 원자적 처리:**
> 1. ACTIVE_BOOKING DELETE (좌석 AVAILABLE 복원)
> 2. BOOKING → CANCELLED
> 3. ORDER → CANCELLED
> 4. [쿠폰 적용 시] USER_COUPON → ISSUED 복원 (재사용 가능)
> 5. Mock PG 환불 요청

**Response 200 OK**
```json
{
  "message": "주문이 취소되었습니다.",
  "orderId": 101,
  "refundAmount": 325000
}
```

**Error Cases**
```json
// 400 — 취소 기간 지남
{ "status": 400, "code": "CANCEL_PERIOD_EXPIRED", "message": "취소 가능 기간이 지났습니다. (이벤트 시작 24시간 전까지)" }

// 400 — 이미 취소됨
{ "status": 400, "code": "ORDER_ALREADY_CANCELLED", "message": "이미 취소된 주문입니다." }

// 403 — 본인 주문 아님
{ "status": 403, "code": "ORDER_NOT_OWNED", "message": "본인의 주문만 취소할 수 있습니다." }
```

---

## 7. 쿠폰 (Coupons)

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| POST | `/api/admin/coupons` | 쿠폰 캠페인 등록 | `{ name, discountAmount, totalQuantity, startAt, expiredAt }` | `201 { couponId }` | 🔐👑 | UC-011 |
| POST | `/api/coupons/{couponId}/issue` | 선착순 쿠폰 발급 ⚠️ | — | `200 { userCouponId }` | 🔐 | UC-011 |

### POST /api/admin/coupons

**Request**
```json
{
  "name": "신규 가입 5,000원 할인",
  "discountAmount": 5000,
  "totalQuantity": 100,
  "startAt": "2026-04-10T12:00:00",
  "expiredAt": "2026-05-31T23:59:59"
}
```

**Response 201 Created**
```json
{
  "couponId": 3,
  "name": "신규 가입 5,000원 할인",
  "totalQuantity": 100,
  "remainingQuantity": 100,
  "startAt": "2026-04-10T12:00:00",
  "expiredAt": "2026-05-31T23:59:59"
}
```

---

### POST /api/coupons/{couponId}/issue ⚠️

> **동시성 제어:** Redis Atomic DECR + Lua Script (수량 0 이하면 즉시 차단 — DECR 자체가 원자적이므로 추가 분산락 불필요)
> **MySQL `remaining_quantity`:** Redis 유실 대비 Source of Truth. Redis 장애 시 DB `SELECT ... FOR UPDATE`로 fallback
> **실패 전략:** Retry with backoff 최대 3회 (쿠폰은 "받느냐 못 받느냐"가 UX 핵심 → 좌석과 달리 재시도 유리)
> **중복 방지:** `user_coupon (user_id, coupon_id)` UNIQUE 제약 + 발급 전 존재 여부 확인 (락 획득 전 선행 체크)

**Response 200 OK**
```json
{
  "message": "쿠폰이 발급되었습니다!",
  "userCouponId": 10,
  "couponName": "신규 가입 5,000원 할인",
  "discountAmount": 5000,
  "expiredAt": "2026-05-31T23:59:59"
}
```

**Error Cases**
```json
// 409 — 이미 발급받음
{ "status": 409, "code": "COUPON_ALREADY_ISSUED", "message": "이미 발급받은 쿠폰입니다." }

// 409 — 수량 소진
{ "status": 409, "code": "COUPON_EXHAUSTED", "message": "쿠폰이 모두 소진되었습니다." }

// 400 — 발급 시작 전
{ "status": 400, "code": "COUPON_NOT_STARTED", "message": "쿠폰 발급 시간이 아닙니다. (발급 시작: 2026-04-10T12:00:00)" }

// 503 — 락 획득 최종 실패 (3회 재시도 후)
{ "status": 503, "code": "SERVICE_UNAVAILABLE", "message": "요청이 많아 처리할 수 없습니다. 잠시 후 다시 시도해주세요." }
```

---

## 8. CS 채팅 (Chat) — REST

| Method | Endpoint | 설명 | Request | Response | 인증 | 관련 UC |
|--------|----------|------|---------|----------|------|---------|
| POST | `/api/chat/rooms` | 채팅방 생성 (또는 기존 방 반환) | — | `200 { chatRoomId }` | 🔐 | UC-012 |
| GET | `/api/chat/rooms/{chatRoomId}/messages` | 채팅 이력 조회 | `?cursor&size` | `200 List<ChatMessage>` | 🔐 | UC-012 |
| PATCH | `/api/chat/rooms/{chatRoomId}/close` | 채팅방 종료 | — | `200` | 🔐 | UC-012 |
| GET | `/api/admin/chat/rooms` | 관리자용 채팅방 목록 조회 | `?status&page&size` | `200 Page<ChatRoom>` | 🔐👑 | UC-012, UC-013 |
| PATCH | `/api/admin/bookings/{bookingId}/confirm` | 예매 수동 확정 (Admin) ⚠️ | — | `200 { bookingId, status }` | 🔐👑 | UC-013 |

### POST /api/chat/rooms

> 이미 OPEN 상태의 채팅방이 있으면 신규 생성 없이 기존 방 ID 반환 (중복 생성 방지)

**Response 200 OK** (신규 생성)
```json
{
  "chatRoomId": 55,
  "status": "OPEN",
  "createdAt": "2026-04-09T20:15:00",
  "isNew": true
}
```

**Response 200 OK** (기존 방 반환)
```json
{
  "chatRoomId": 55,
  "status": "OPEN",
  "createdAt": "2026-04-09T20:10:00",
  "isNew": false
}
```

---

### GET /api/chat/rooms/{chatRoomId}/messages

> **커서 기반 페이징:** `cursor` = 마지막으로 받은 `chatMessageId`, `size` 기본값 50
> 인덱스: `(chat_room_id, chat_message_id DESC)`
> 본인 채팅방 또는 ADMIN만 접근 가능

**Query Parameters:** `?cursor=200&size=50`

**Response 200 OK**
```json
{
  "chatRoomId": 55,
  "messages": [
    {
      "chatMessageId": 151,
      "senderId": 1,
      "senderRole": "USER",
      "senderNickname": "이도현",
      "content": "결제는 됐는데 예매 내역에 티켓이 없어요. 주문번호 101입니다.",
      "sentAt": "2026-04-09T20:15:10"
    },
    {
      "chatMessageId": 152,
      "senderId": 99,
      "senderRole": "ADMIN",
      "senderNickname": "TicketFlow CS팀",
      "content": "확인 중입니다. 결제 확인되어 수동 확정 처리하겠습니다.",
      "sentAt": "2026-04-09T20:18:05"
    }
  ],
  "nextCursor": 151,
  "hasNext": true
}
```

---

### PATCH /api/chat/rooms/{chatRoomId}/close

**Response 200 OK**
```json
{
  "message": "채팅방이 종료되었습니다.",
  "chatRoomId": 55,
  "closedAt": "2026-04-09T20:25:00"
}
```

---

### GET /api/admin/chat/rooms

> 관리자 대시보드 초기 로딩 시 현재 진행 중인 채팅방 목록을 조회한다.
> 신규 채팅방은 WebSocket `/sub/chat/rooms` push로 수신. 이 REST API는 **초기 데이터 로딩 전용**이다.
> ADMIN 권한 필수.

**Query Parameters:** `?status=OPEN&page=0&size=20`

**Response 200 OK**
```json
{
  "content": [
    {
      "chatRoomId": 55,
      "userId": 1,
      "userNickname": "이도현",
      "status": "OPEN",
      "lastMessage": "결제는 됐는데 예매 내역에 티켓이 없어요.",
      "createdAt": "2026-04-09T20:15:00"
    },
    {
      "chatRoomId": 54,
      "userId": 2,
      "userNickname": "박서준",
      "status": "OPEN",
      "lastMessage": "좌석 선택이 안 됩니다.",
      "createdAt": "2026-04-09T19:55:00"
    }
  ],
  "page": 0,
  "size": 20,
  "totalElements": 2,
  "hasNext": false
}
```

**Error Cases**
```json
// 403 — ADMIN 권한 없음
{ "status": 403, "code": "FORBIDDEN", "message": "관리자 권한이 필요합니다." }
```

---

### PATCH /api/admin/bookings/{bookingId}/confirm ⚠️

> **동시성 제어:** `SELECT ... FOR UPDATE`로 BOOKING, ORDER 행을 잠금 (취소 트랜잭션과의 충돌 방지)
> **전제 조건:** PAYMENT 테이블에 해당 orderId의 SUCCESS 결제 기록이 존재해야 함
> **원자적 처리:** ORDER/BOOKING → CONFIRMED, ACTIVE_BOOKING INSERT (단일 트랜잭션)
> ADMIN 권한 필수.

**Response 200 OK**
```json
{
  "message": "수동 확정이 완료되었습니다.",
  "bookingId": 201,
  "orderId": 101,
  "status": "CONFIRMED",
  "ticketCode": "TF-2026-A015-XYZ1",
  "confirmedAt": "2026-04-09T20:20:00"
}
```

**Error Cases**
```json
// 400 — 결제 기록 없음
{ "status": 400, "code": "PAYMENT_NOT_FOUND", "message": "결제 기록이 확인되지 않습니다. 확정을 중단합니다." }

// 400 — 이미 확정/취소된 예매
{ "status": 400, "code": "BOOKING_NOT_PENDING", "message": "PENDING 상태의 예매만 수동 확정할 수 있습니다." }

// 409 — 해당 좌석이 이미 다른 예매로 확정됨 (ACTIVE_BOOKING PK 충돌)
{ "status": 409, "code": "SEAT_ALREADY_CONFIRMED", "message": "이미 해당 좌석이 다른 예매로 확정되어 있습니다." }

// 403 — ADMIN 권한 없음
{ "status": 403, "code": "FORBIDDEN", "message": "관리자 권한이 필요합니다." }
```

---

## 9. CS 채팅 — WebSocket (STOMP)

> **연결 엔드포인트:** `ws://api.ticketflow.io/ws-stomp`
> **프로토콜:** STOMP over WebSocket
> **인증:** STOMP CONNECT 프레임의 `Authorization` 헤더에 JWT 포함
> **브로커:** Spring 내장 Simple Broker (외부 메시지 브로커 불필요)

| 구분 | 경로 | 설명 |
|------|------|------|
| 연결 | `ws://api.ticketflow.io/ws-stomp` | WebSocket 핸드셰이크 |
| 구독 (사용자) | `/sub/chat/room/{chatRoomId}` | 해당 채팅방 메시지 수신 |
| 구독 (관리자) | `/sub/chat/rooms` | 신규 채팅방 알림 수신 |
| 메시지 전송 | `/pub/chat/message` | 메시지 발행 (사용자/관리자 동일) |

### STOMP CONNECT 프레임
```
CONNECT
Authorization:Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
accept-version:1.1,1.0
heart-beat:10000,10000
```

### 메시지 전송 (/pub/chat/message)

**Payload**
```json
{
  "chatRoomId": 55,
  "content": "결제는 됐는데 예매 내역에 티켓이 없어요."
}
```

### 메시지 수신 (/sub/chat/room/{chatRoomId})

**Broadcast Payload** (서버 → 구독자)
```json
{
  "chatMessageId": 151,
  "chatRoomId": 55,
  "senderId": 1,
  "senderRole": "USER",
  "senderNickname": "이도현",
  "content": "결제는 됐는데 예매 내역에 티켓이 없어요.",
  "sentAt": "2026-04-09T20:15:10"
}
```

### 신규 채팅방 알림 (/sub/chat/rooms) — 관리자용

```json
{
  "chatRoomId": 55,
  "userId": 1,
  "userNickname": "이도현",
  "openedAt": "2026-04-09T20:15:00"
}
```

---

## 10. API 전체 요약 인덱스

| # | Method | Endpoint | 설명 | 인증 | 특이사항 |
|---|--------|----------|------|------|----------|
| 1 | POST | `/api/auth/signup` | 회원가입 | 🔓 | |
| 2 | POST | `/api/auth/login` | 로그인 | 🔓 | |
| 3 | GET | `/api/users/me` | 내 정보 조회 | 🔐 | |
| 4 | PATCH | `/api/users/me` | 내 정보 수정 | 🔐 | |
| 5 | GET | `/api/users/me/bookings` | 내 예매 내역 | 🔐 | |
| 6 | GET | `/api/users/me/coupons` | 내 쿠폰 목록 | 🔐 | |
| 7 | POST | `/api/admin/events` | 이벤트 등록 | 🔐👑 | |
| 8 | GET | `/api/events` | 이벤트 목록 조회 | 🔓 | 💾 Caffeine |
| 9 | GET | `/api/events/{eventId}` | 이벤트 상세 조회 | 🔓 | 💾 Caffeine |
| 10 | GET | `/api/v1/events/search` | 검색 v1 (캐시 없음) | 🔓 | |
| 11 | GET | `/api/v2/events/search` | 검색 v2 (캐시) | 🔓 | 💾 Caffeine→Redis |
| 12 | GET | `/api/search/popular` | 인기 검색어 Top 10 | 🔓 | 💾 Redis ZSet |
| 13 | GET | `/api/events/{eventId}/seats` | 좌석 상태 조회 | 🔐 | |
| 14 | POST | `/api/events/{eventId}/seats/{seatId}/hold` | 좌석 Hold | 🔐 | ⚠️ 분산락 |
| 15 | DELETE | `/api/events/{eventId}/seats/{seatId}/hold` | Hold 해제 | 🔐 | |
| 16 | POST | `/api/orders` | 주문 생성 (예매) | 🔐 | ⚠️ 쿠폰 락 |
| 17 | POST | `/api/mock-pg/webhook` | PG 웹훅 수신 | 🔓 | ⚠️ 멱등성 보장 |
| 18 | POST | `/api/orders/{orderId}/cancel` | 주문 취소 | 🔐 | |
| 19 | POST | `/api/admin/coupons` | 쿠폰 등록 | 🔐👑 | |
| 20 | POST | `/api/coupons/{couponId}/issue` | 쿠폰 발급 | 🔐 | ⚠️ Redis Atomic |
| 21 | POST | `/api/chat/rooms` | 채팅방 생성 | 🔐 | |
| 22 | GET | `/api/chat/rooms/{chatRoomId}/messages` | 채팅 이력 | 🔐 | 커서 페이징 |
| 23 | PATCH | `/api/chat/rooms/{chatRoomId}/close` | 채팅방 종료 | 🔐 | |
| WS | STOMP | `ws://.../ws-stomp` | 실시간 채팅 | 🔐 JWT | STOMP 프로토콜 |
| 24 | POST | `/api/search/popular/click` | 인기 검색어 클릭 점수 반영 | 🔓 | ZINCRBY |
| 25 | GET | `/api/admin/chat/rooms` | 관리자 채팅방 목록 조회 | 🔐👑 | 초기 로딩용 |
| 26 | PATCH | `/api/admin/bookings/{bookingId}/confirm` | 예매 수동 확정 | 🔐👑 | ⚠️ FOR UPDATE |

---

## 11. 주요 설계 결정 요약

### 페이징 전략
- **이벤트 목록/검색:** offset 기반 (`page`, `size`) — 총 페이지 수 표시가 필요한 탐색형 UI
- **채팅 이력:** cursor 기반 (`cursor`, `size`) — 무한 스크롤, 실시간 메시지 추가 환경에서 offset 사용 시 데이터 누락/중복 발생 위험
- **내 예매 내역:** offset 기반 — 사용자가 직접 페이지를 선택하는 패턴

### 동시성 민감 API ⚠️ 설계 원칙

| API | 제어 방식 | 실패 전략 | 이유 |
|-----|-----------|-----------|------|
| 좌석 Hold | Redis SETNX 분산락 (TTL 3초) | Fail Fast (즉시 409) | 티켓팅은 "빠른 응답"이 UX 핵심. 기다려도 다른 사람이 이미 선택함 |
| 쿠폰 발급 | Redis Atomic DECR + Lua | Retry 3회 Backoff | "받느냐 못 받느냐"가 UX 핵심. 잠깐 기다려서라도 받는 게 유리 |
| 쿠폰 사용 | 분산락 + `@Version` 낙관적 락 | 예외 발생 시 트랜잭션 롤백 | 쿠폰 1장 사용은 충돌 빈도 낮음. DB 트랜잭션 범위 내 처리 충분 |
| 웹훅 확정 | 멱등성 키 (`orderId`) | 중복 수신 시 200 즉시 반환 | PG는 웹훅을 여러 번 보낼 수 있음. 중복 처리 방지 필수 |
| 수동 예매 확정 | `SELECT ... FOR UPDATE` 비관적 락 | 409 SEAT_ALREADY_CONFIRMED | 관리자 수동 처리 중 자동 취소 트랜잭션과의 충돌 방지 |
| 주문 취소 | `SELECT ... FOR UPDATE` 비관적 락 | 단일 트랜잭션 내 ACTIVE_BOOKING DELETE → BOOKING/ORDER CANCELLED | 취소 중 다른 새 예매 확정 트랜잭션과의 충돌 방지 |

### REST 원칙 준수 포인트
- 취소는 `DELETE /api/orders/{orderId}` 대신 `POST /api/orders/{orderId}/cancel` — 취소는 상태 전이(CANCELLED)를 유발하는 복잡한 비즈니스 로직(PG 환불, 좌석 복원 등)을 수반하므로, 단순 삭제(`DELETE`)로 표현하기 부적절
- Hold 해제는 `DELETE /api/events/{eventId}/seats/{seatId}/hold` — Hold 리소스 자체를 삭제하는 의미이므로 DELETE 적합

---

## 다음 문서 연결

- **7. 화면 설계서** — 본 API 명세서의 Request/Response를 기반으로 각 화면의 데이터 바인딩 명시
- **8. 인프라 아키텍처 다이어그램** — API Gateway, EC2, RDS, ElastiCache 구성과 API 트래픽 흐름
- **9. 동시성 제어 설계서** — ⚠️ 표시된 API의 Race Condition 시나리오 매트릭스 상세화
- **10. ADR** — 커서 vs. offset 페이징 선택 이유, 취소 API 설계 결정 등 기록
