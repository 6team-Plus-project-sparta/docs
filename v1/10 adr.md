# 📝 ADR (Architecture Decision Records)

> 각 ADR은 하나의 기술 결정을 다룹니다.
> 형식: 컨텍스트 → 결정 → 근거 → Trade-off → 결과

---

## ADR-001: 인증 방식 — JWT (Stateless)

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정

### 컨텍스트
사용자 인증이 필요하다. 세션 기반과 토큰 기반 중 선택해야 한다.

### 결정
**JWT (Access Token 15분 + Refresh Token 7일)** 방식 채택

### 근거 및 Trade-off

| 방식 | 확장성 | 구현 복잡도 | 보안 | 팀 학습 여부 |
|------|--------|------------|------|-------------|
| 세션 (서버 저장) | ❌ Scale-out 시 세션 공유 필요 | 낮음 | 높음 | 알고 있음 |
| **JWT** | ✅ Stateless, 수평 확장 용이 | 중간 | 중간 | 배우는 중 |

- Access Token을 짧게(15분) 설정해 탈취 피해 최소화
- Refresh Token은 Redis에 저장해 강제 로그아웃(블랙리스트) 구현 가능
- AWS EC2 다중 서버 배포 시 세션 공유 없이도 인증 가능

### 결과
`spring-security-oauth2-resource-server` 또는 `jjwt` 라이브러리로 구현

---

## ADR-002: 로컬 캐시 라이브러리 — Caffeine

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정

### 컨텍스트
검색 API 성능 개선을 위한 In-memory 캐시가 필요하다.

### 결정
**Caffeine** 채택 (`spring-boot-starter-cache` + `@Cacheable`)

### 근거 및 Trade-off

| 라이브러리 | 성능 | Spring 연동 | TTL 지원 | 비고 |
|-----------|------|------------|----------|------|
| ConcurrentHashMap | ⭐⭐ | 수동 구현 | ❌ | 직접 관리 필요 |
| Ehcache | ⭐⭐⭐ | ✅ | ✅ | 설정 복잡 |
| **Caffeine** | ⭐⭐⭐⭐ | ✅ Spring AOP | ✅ TTL + maximumSize | W-TinyLFU 알고리즘, 최고 성능 |

- `@Cacheable` 어노테이션 하나로 AOP 방식 적용 → 비즈니스 코드 침범 없음
- TTL + maximumSize로 메모리 폭증 방지
- 단점: 서버 다중화 시 서버 간 캐시 불일치 → 도전 기능에서 Redis Cache로 전환

---

## ADR-003: 동시성 제어 — Redis 분산락 (Lettuce SETNX)

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정 (Redisson은 도전 기능)

### 컨텍스트
티켓 예매·쿠폰 발급의 동시성 문제를 해결해야 한다.

### 결정
**Lettuce + SETNX + TTL + Lua Script** 방식으로 직접 구현  
*(과제 요구사항: 필수는 Lettuce, 도전은 Redisson)*

### 근거 및 Trade-off

| 방식 | 구현 난이도 | 기능 | 재시도 | 팀 학습 가치 |
|------|------------|------|--------|-------------|
| DB 비관적 락 | 낮음 | DB 범위만 | ❌ | 낮음 |
| DB 낙관적 락 | 낮음 | 충돌 감지 | 수동 구현 | 중간 |
| **Lettuce SETNX** | 중간 | 비즈니스 전체 | 수동 구현 | **높음** (원리 학습) |
| Redisson | 낮음 | 비즈니스 전체 + Pub/Sub 재시도 | ✅ 자동 | 중간 (추상화 높음) |

- Lettuce는 기존 Spring Data Redis 의존성 안에 포함 → 추가 의존성 불필요
- SETNX + UUID + Lua Script를 직접 구현함으로써 분산락 원리를 깊이 이해
- Redisson은 도전 기능으로 구현해 두 방식을 비교

---

## ADR-004: 인기 검색어 구현 — Redis Sorted Set

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정

### 컨텍스트
인기 검색어 Top 10을 집계하고 빠르게 조회해야 한다.

### 결정
**Redis Sorted Set (ZSet)** 사용
- Key: `popular:search`
- 검색 시마다: `ZINCRBY popular:search 1 {keyword}`
- 조회 시: `ZREVRANGE popular:search 0 9 WITHSCORES`

### 근거

| 방식 | 실시간성 | 구현 난이도 | DB 부하 |
|------|----------|------------|---------|
| DB COUNT GROUP BY | ❌ 느림 | 낮음 | ❌ 높음 |
| RDB 별도 집계 테이블 | 중간 | 높음 | 중간 |
| **Redis ZSet** | ✅ 실시간 | 낮음 | ✅ 없음 |

- ZSet의 ZINCRBY는 O(log N) 원자적 연산 → 동시성 문제 없음
- ZREVRANGE로 상위 N개 즉시 조회 → O(log N + N)
- Caffeine 캐시로 1차 감쇠: TTL 5분 내 반복 조회는 Redis도 조회하지 않음

---

## ADR-005: 검색 API — QueryDSL 동적 쿼리

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정

### 컨텍스트
여러 검색 조건(keyword, genre, 날짜)의 조합을 처리해야 한다.

### 결정
**QueryDSL + BooleanExpression** 방식 채택

### 근거

| 방식 | 타입 안전 | 동적 조건 | 복잡한 쿼리 |
|------|----------|----------|------------|
| JPQL 문자열 | ❌ | 어려움 | 어려움 |
| Spring Data JPA Specification | ✅ | ✅ | 중간 |
| **QueryDSL BooleanExpression** | ✅ | ✅ 우수 | ✅ 우수 |
| Native Query | N/A | ✅ | ✅ | 유지보수 어려움 |

```java
// BooleanExpression 조합 예시
private BooleanExpression titleContains(String keyword) {
    return StringUtils.hasText(keyword) 
        ? event.title.containsIgnoreCase(keyword) 
        : null; // null이면 QueryDSL이 자동으로 조건 제외
}
```

null 반환 시 조건이 자동 무시되어 Optional 체이닝 없이 동적 쿼리 구성 가능

---

## ADR-006: 채팅 프로토콜 — STOMP over WebSocket

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정

### 컨텍스트
CS 문의 채팅의 실시간 양방향 통신이 필요하다.

### 결정
**순수 WebSocket → STOMP 프로토콜** 적용

### 근거

| 방식 | 구현 복잡도 | 메시지 라우팅 | Spring 지원 | 채팅방 분리 |
|------|------------|--------------|------------|------------|
| HTTP Long Polling | 낮음 | ❌ | ✅ | ❌ |
| SSE (단방향) | 낮음 | ❌ 서버→클라만 | ✅ | ❌ |
| 순수 WebSocket | 중간 | 수동 구현 | ✅ | 수동 |
| **STOMP** | 중간 | ✅ pub/sub | ✅ Spring STOMP | `/sub/chat/{roomId}` |

- STOMP의 SUBSCRIBE/SEND 구조로 채팅방별 메시지 자동 라우팅
- Spring의 `@MessageMapping` + `SimpMessagingTemplate`으로 간결한 서버 코드
- ChannelInterceptor에서 JWT 검증으로 WebSocket 보안 처리

---

## ADR-007: 페이징 전략 — 채팅은 커서, 검색은 Offset

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정

### 컨텍스트
채팅 메시지와 이벤트 검색 각각에 맞는 페이징 전략이 필요하다.

### 결정
- **채팅 메시지**: 커서 기반 페이징 (`WHERE id < :lastMessageId`)
- **이벤트 검색**: Offset 기반 페이징 (`Page<>`, `Pageable`)

### 근거

| 방식 | 일관성 | 중간 삽입 시 | 임의 페이지 이동 | 적합 용도 |
|------|--------|------------|-----------------|-----------|
| Offset | ❌ (새 데이터 삽입 시 중복/누락) | ❌ | ✅ | **검색 결과** (임의 페이지 이동 필요) |
| **Cursor** | ✅ | ✅ | ❌ | **채팅** (시간순 스크롤, 삽입 빈번) |

- 채팅: 메시지가 실시간으로 추가되므로 Offset은 중복 노출 문제 발생
- 검색: "3페이지로 이동" 같은 임의 탐색이 필요하므로 Offset이 적합

---

## ADR-008: Mock PG 결제 설계

**날짜:** 프로젝트 설계 1일차  
**상태:** 확정

### 컨텍스트
3주 일정 안에 실제 PG 연동은 불가능하지만 포트폴리오 임팩트를 유지하고 싶다.

### 결정
**토스페이먼츠 API 구조를 모방한 Mock PG** 구현

### 구현 방식
```java
@Component
public class MockTossPaymentClient {
    public PaymentResult confirm(String orderId, int amount, String method) {
        // 95% 확률 성공, 5% 실패 (랜덤)
        if (Math.random() < 0.95) {
            return PaymentResult.success("TXN-" + UUID.randomUUID());
        }
        return PaymentResult.failure("PAYMENT_FAILED");
    }
}
```

### 이점
- 실제 결제 API URL·Request/Response 구조를 코드에 반영 → 포트폴리오에서 설명 가능
- 추후 실제 PG 연동 시 `MockTossPaymentClient` → `RealTossPaymentClient`로 교체만 하면 됨 (전략 패턴)
- 테스트 환경에서 카드 번호 노출 없이 결제 플로우 전체 테스트 가능