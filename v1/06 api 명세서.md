# 📡 API 명세서

## 공통 규칙

| 항목 | 내용 |
|------|------|
| Base URL | `https://api.ticketflow.com` |
| 인증 방식 | Bearer Token (JWT) — Authorization 헤더 |
| 응답 형식 | `application/json` |
| 날짜 형식 | ISO 8601 (`2025-07-01T14:00:00`) |
| 페이징 | `page`(0-based), `size` 파라미터 |

### 공통 에러 응답

```json
{
  "status": 400,
  "error": "BAD_REQUEST",
  "message": "에러 상세 메시지",
  "timestamp": "2025-07-01T14:00:05"
}
```

---

## 1. 회원 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/api/v1/users/signup` | 회원가입 | ❌ |
| POST | `/api/v1/users/login` | 로그인 | ❌ |
| POST | `/api/v1/users/refresh` | 토큰 재발급 | ❌ |
| GET | `/api/v1/users/me` | 내 정보 조회 | ✅ |
| PUT | `/api/v1/users/me` | 내 정보 수정 | ✅ |

### POST /api/v1/users/signup

**Request**
```json
{
  "email": "user@example.com",
  "password": "Password1!",
  "nickname": "지수",
  "phone": "010-1234-5678"
}
```

**Response 201**
```json
{
  "userId": 1,
  "email": "user@example.com",
  "nickname": "지수"
}
```

### POST /api/v1/users/login

**Request**
```json
{ "email": "user@example.com", "password": "Password1!" }
```

**Response 200**
```json
{
  "accessToken": "eyJhbGci...",
  "refreshToken": "eyJhbGci...",
  "tokenType": "Bearer"
}
```

---

## 2. 이벤트 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/api/v1/events` | 이벤트 목록 조회 | ❌ |
| GET | `/api/v1/events/{eventId}` | 이벤트 상세 조회 | ❌ |
| GET | `/api/v1/events/search` | 이벤트 검색 v1 (캐시 없음) | ❌ |
| GET | `/api/v2/events/search` | 이벤트 검색 v2 (캐시 적용) | ❌ |
| GET | `/api/v1/events/popular-keywords` | 인기 검색어 Top10 | ❌ |
| POST | `/api/v1/admin/events` | 이벤트 등록 | ✅ Admin |
| PUT | `/api/v1/admin/events/{eventId}` | 이벤트 수정 | ✅ Admin |
| DELETE | `/api/v1/admin/events/{eventId}` | 이벤트 삭제 | ✅ Admin |

### GET /api/v1/events/search (v1)

**Query Parameters**

| 파라미터 | 타입 | 필수 | 설명 |
|---------|------|------|------|
| keyword | String | ❌ | 이벤트명 LIKE 검색 |
| genre | String | ❌ | CONCERT, MUSICAL, PLAY, BASEBALL, SOCCER, BASKETBALL |
| startDate | String | ❌ | 검색 시작일 (yyyy-MM-dd) |
| endDate | String | ❌ | 검색 종료일 (yyyy-MM-dd) |
| page | int | ❌ | 기본값 0 |
| size | int | ❌ | 기본값 10 |

**Response 200**
```json
{
  "content": [
    {
      "eventId": 1,
      "title": "아이유 콘서트 2025",
      "genre": "CONCERT",
      "venue": "올림픽공원 체조경기장",
      "startAt": "2025-08-15T19:00:00",
      "posterImageUrl": "https://cdn.ticketflow.com/...",
      "status": "ON_SALE",
      "minPrice": 77000
    }
  ],
  "page": 0,
  "size": 10,
  "totalElements": 42,
  "totalPages": 5
}
```

### GET /api/v1/events/popular-keywords

**Response 200**
```json
{
  "keywords": [
    { "rank": 1, "keyword": "아이유", "score": 1523 },
    { "rank": 2, "keyword": "BTS", "score": 1201 },
    { "rank": 3, "keyword": "한화이글스", "score": 987 }
  ],
  "updatedAt": "2025-07-01T14:00:00"
}
```

---

## 3. 예매 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/api/v1/events/{eventId}/sections` | 구역별 잔여 좌석 조회 | ❌ |
| GET | `/api/v1/sections/{sectionId}/seats` | 좌석 상태 조회 | ❌ |
| POST | `/api/v1/seats/{seatId}/hold` | 좌석 임시 선점 | ✅ |
| POST | `/api/v1/bookings` | 예매 요청 | ✅ |
| POST | `/api/v1/bookings/{bookingId}/pay` | 결제 요청 (Mock PG) | ✅ |
| GET | `/api/v1/bookings/me` | 내 예매 내역 조회 | ✅ |
| DELETE | `/api/v1/bookings/{bookingId}` | 예매 취소 | ✅ |

### POST /api/v1/bookings

**Request**
```json
{
  "seatId": 101,
  "couponIssueId": 5
}
```

**Response 201**
```json
{
  "bookingId": 200,
  "seatInfo": {
    "eventTitle": "아이유 콘서트 2025",
    "sectionName": "A구역",
    "row": "C",
    "seatNumber": 15
  },
  "originalAmount": 154000,
  "discountAmount": 30800,
  "finalAmount": 123200,
  "status": "PENDING",
  "expiredAt": "2025-07-01T14:07:00"
}
```

### POST /api/v1/bookings/{bookingId}/pay

**Request**
```json
{
  "paymentMethod": "CARD",
  "cardNumber": "****-****-****-1234"
}
```

**Response 200**
```json
{
  "paymentId": 300,
  "bookingId": 200,
  "status": "SUCCESS",
  "paidAt": "2025-07-01T14:02:30"
}
```

---

## 4. 쿠폰 API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| GET | `/api/v1/coupons` | 발급 가능한 쿠폰 목록 | ❌ |
| POST | `/api/v1/coupons/{couponId}/issue` | 선착순 쿠폰 발급 | ✅ |
| GET | `/api/v1/coupons/me` | 내 쿠폰 목록 | ✅ |
| POST | `/api/v1/admin/coupons` | 쿠폰 등록 | ✅ Admin |

### POST /api/v1/coupons/{couponId}/issue

**Response 201**
```json
{
  "couponIssueId": 5,
  "couponName": "여름 특가 쿠폰",
  "discountType": "PERCENT",
  "discountValue": 20,
  "expiredAt": "2025-07-31T23:59:59"
}
```

**Response 400 (소진)**
```json
{
  "status": 400,
  "error": "COUPON_EXHAUSTED",
  "message": "쿠폰이 모두 소진되었습니다."
}
```

---

## 5. CS 채팅 API

### HTTP REST API

| Method | URI | 설명 | 인증 |
|--------|-----|------|------|
| POST | `/api/v1/chat/rooms` | 문의 채팅방 생성 | ✅ User |
| GET | `/api/v1/chat/rooms/me` | 내 문의 목록 | ✅ User |
| GET | `/api/v1/chat/rooms` | 전체 문의 목록 (상태 필터) | ✅ Admin |
| GET | `/api/v1/chat/rooms/{roomId}/messages` | 채팅 내역 조회 (커서) | ✅ |
| PATCH | `/api/v1/chat/rooms/{roomId}/status` | 문의 상태 변경 | ✅ Admin |

### GET /api/v1/chat/rooms/{roomId}/messages

**Query Parameters:** `lastMessageId` (optional), `size` (default 20)

**Response 200**
```json
{
  "messages": [
    {
      "messageId": 50,
      "senderId": 1,
      "senderNickname": "지수",
      "content": "공연 날짜가 변경되었다는데 환불 가능한가요?",
      "createdAt": "2025-07-01T13:55:00"
    }
  ],
  "hasNext": true,
  "nextCursorId": 49
}
```

### WebSocket (STOMP)

| 타입 | Destination | 설명 |
|------|-------------|------|
| CONNECT | `ws://host/ws-chat` | 연결 (JWT 헤더 필수) |
| SUBSCRIBE | `/sub/chat/{roomId}` | 채팅방 메시지 수신 |
| SEND | `/pub/chat/{roomId}` | 메시지 전송 |

**SEND Payload**
```json
{ "content": "안녕하세요. 문의드립니다." }
```

**수신 메시지 형식**
```json
{
  "messageId": 51,
  "senderId": 1,
  "senderNickname": "지수",
  "content": "안녕하세요. 문의드립니다.",
  "createdAt": "2025-07-01T14:00:01"
}
```

---

## 6. OpenAPI YAML (핵심 API 발췌)

```yaml
openapi: 3.0.3
info:
  title: TicketFlow API
  version: 1.0.0
  description: 공연·스포츠 티켓팅 플랫폼 백엔드 API

servers:
  - url: https://api.ticketflow.com
    description: Production
  - url: http://localhost:8080
    description: Local

security:
  - BearerAuth: []

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

paths:
  /api/v1/users/login:
    post:
      tags: [Auth]
      summary: 로그인
      security: []
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                email:
                  type: string
                  format: email
                password:
                  type: string
      responses:
        '200':
          description: 로그인 성공
          content:
            application/json:
              schema:
                type: object
                properties:
                  accessToken:
                    type: string
                  refreshToken:
                    type: string
        '401':
          description: 인증 실패

  /api/v1/events/search:
    get:
      tags: [Event]
      summary: 이벤트 검색 v1 (캐시 없음)
      security: []
      parameters:
        - name: keyword
          in: query
          schema:
            type: string
        - name: genre
          in: query
          schema:
            type: string
            enum: [CONCERT, MUSICAL, PLAY, BASEBALL, SOCCER, BASKETBALL]
        - name: page
          in: query
          schema:
            type: integer
            default: 0
        - name: size
          in: query
          schema:
            type: integer
            default: 10
      responses:
        '200':
          description: 검색 결과

  /api/v2/events/search:
    get:
      tags: [Event]
      summary: 이벤트 검색 v2 (Caffeine 캐시 적용)
      security: []
      parameters:
        - name: keyword
          in: query
          schema:
            type: string
        - name: genre
          in: query
          schema:
            type: string
        - name: page
          in: query
          schema:
            type: integer
            default: 0
        - name: size
          in: query
          schema:
            type: integer
            default: 10
      responses:
        '200':
          description: 캐시 적용 검색 결과

  /api/v1/bookings:
    post:
      tags: [Booking]
      summary: 예매 요청 (분산락 적용)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [seatId]
              properties:
                seatId:
                  type: integer
                  format: int64
                couponIssueId:
                  type: integer
                  format: int64
                  nullable: true
      responses:
        '201':
          description: 예매 요청 성공
        '400':
          description: 재고 없음 / 쿠폰 오류
        '409':
          description: 락 충돌 (재시도 요청)

  /api/v1/coupons/{couponId}/issue:
    post:
      tags: [Coupon]
      summary: 선착순 쿠폰 발급 (분산락 적용)
      parameters:
        - name: couponId
          in: path
          required: true
          schema:
            type: integer
            format: int64
      responses:
        '201':
          description: 쿠폰 발급 성공
        '400':
          description: 소진 / 중복 발급
        '409':
          description: 락 충돌
```