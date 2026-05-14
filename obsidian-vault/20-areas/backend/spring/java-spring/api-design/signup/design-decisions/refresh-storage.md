---
title: "refresh 토큰 저장소 — RDB vs Redis"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - refresh-token
  - redis
---

# refresh 토큰 저장소 — RDB vs Redis

**[[design-decisions|↑ design-decisions hub]]**

> "refresh 토큰을 어디 저장하나" — 잘못 선택하면 **latency 폭증 또는 audit 불가능**.

---

## 1. 본 vault 결정

- **시작: RDB (PostgreSQL `refresh_tokens` 테이블).**
- **성장 후: Redis 마이그레이션 또는 hybrid (RDB 가 진실, Redis 캐시).**
- audit / rotation chain / reuse detection 모두 SQL 로 처리.

---

## 2. 비교 — RDB vs Redis vs Hybrid

### 2.1 RDB (본 vault 시작)

**왜 적합한 케이스**
- 중소 SaaS (MAU 1000만 이하) — DB 부담 무시 가능.
- 강한 audit 필요 — rotation chain / reuse detection / 분쟁 시 입증.
- 인프라 단순화 — Redis 추가 운영 X.

**왜 안 되는 케이스 (대형 SaaS)**
- 분당 100만+ 토큰 검증 → DB 부담 ↑.
- 짧은 TTL (5분 이하) → cleanup 부담 ↑.
- 글로벌 다중 region — DB write 지연.

**안 하면 무슨 문제 (RDB 없이 Redis 만)**
- Redis persistence (AOF / RDB) 의존 — 장애 시 토큰 일부 손실.
- audit 부실 — Redis 의 key-value 한계.
- 분쟁 시 "이 토큰이 언제 어떻게 발급됐나" 입증 어려움.

**대안과 왜 안 됨**
- in-memory (JVM heap) — 다중 인스턴스 동기화 X. 절대 X.
- 파일 시스템 — IO 부담 + 분산 X.

**트레이드오프**
- latency ms 단위 (Redis 의 μs 대비 ↑).
- DB 부담 — 토큰 검증이 매 API 호출의 일부면 (refresh endpoint 만) 부담 적음.

---

### 2.2 Redis (대형 SaaS)

**왜 적합한 케이스**
- 대형 SaaS (MAU 1000만+) — DB 부담 회피.
- 매우 짧은 TTL (5분) — Redis TTL 자동 처리.
- 글로벌 — Redis cluster 가 DB 보다 region 동기화 ↑.

**왜 안 되는 케이스**
- audit / 분쟁 자료 필요 — Redis 의 한계.
- rotation chain 추적 — SQL 의 recursive CTE 만큼 자연스럽지 않음.
- 인프라 / 운영 부담 — Redis cluster, failover, backup 정책 별도.

**안 하면 무슨 문제 (Redis 만, 영속화 부실)**
- AOF disabled = 장애 시 마지막 fsync 후 토큰 사라짐.
- 모든 user 강제 로그아웃 사태.

**대안 — Redis 영속화 강화**
- AOF + everysec — 1초 단위 fsync. 1초 데이터 손실 가능.
- AOF + always — 매 write fsync. throughput ↓.

**트레이드오프**
- 운영 부담 ↑ vs latency ↓.

---

### 2.3 Hybrid (RDB + Redis cache)

```
[발급 시]
  → RDB INSERT (영속)
  → Redis SET (캐시, TTL = refresh TTL)

[검증 시]
  1. Redis GET — hit 면 즉시 응답
  2. miss 면 RDB lookup + Redis 캐시 채움

[Rotate 시]
  → RDB UPDATE (ROTATED + new INSERT)
  → Redis 새 토큰 SET + 옛 토큰 DEL
```

**왜 적합**
- 중간 규모 (MAU 100만~1000만) — RDB 안정 + Redis 속도.
- audit 는 RDB 에 있음, latency 는 Redis 가 처리.

**복잡도**
- 두 곳 동기화 부담 — cache invalidation 정책 필요.
- "Redis 와 RDB 가 불일치" race condition — RDB 가 진실의 원천.

---

## 3. RDB 스키마

자세히: [[../database/refresh-tokens-table]].

```sql
CREATE TABLE refresh_tokens (
    id CHAR(26) PRIMARY KEY,
    user_id CHAR(26) NOT NULL REFERENCES users(id),
    token_hash CHAR(64) NOT NULL,
    status VARCHAR(20) NOT NULL,
    rotated_to_id CHAR(26),
    issued_at TIMESTAMPTZ,
    expires_at TIMESTAMPTZ,
    ...
);

CREATE UNIQUE INDEX ux_refresh_tokens_hash ON refresh_tokens (token_hash);
```

**왜 RDB 가 이 데이터에 적합**
- relational query (user 의 모든 ACTIVE RT, 도난 chain 추적) 자연.
- ACID — 동시 rotate 시 일관성 보장.
- audit (TIMESTAMPTZ 컬럼들) 풍부.

---

## 4. Redis schema (대형 SaaS / hybrid)

```
Key:   rt:<token_hash>
Value: JSON { id, userId, status, issuedAt, expiresAt, rotatedToId }
TTL:   14d
```

또는 user 별 set:

```
Key:   user:<userId>:rt
Value: SET of token_hash
TTL:   14d (자동 만료)
```

**왜 hash 가 key**
- `WHERE token_hash = ?` 의 lookup 이 매 검증의 80%.
- Redis 의 GET 가 O(1).

**왜 user 별 SET 도**
- "user 의 모든 RT" 조회 (logoutAll) 가 O(1).

---

## 5. 마이그레이션 흐름 — RDB → Redis

### 5.1 단계

```
1. dual-write (RDB + Redis)
   - 모든 발급 / rotate 가 둘 다 write
2. dual-read 시작 (Redis 우선, miss 시 RDB)
3. 모니터링 (cache hit rate)
4. Redis-only read (RDB write 만 유지 - audit)
5. RDB write 도 async (옵션) - latency 더 ↓
```

### 5.2 왜 점진적

- 한 번에 전환 → Redis 의존 장애 시 모든 user 로그아웃.
- dual-write/read 단계에서 hit rate 모니터링 + 회귀 발견.

---

## 6. TTL 관리

### 6.1 RDB

```sql
DELETE FROM refresh_tokens WHERE expires_at < now() - INTERVAL '7 days';
```

→ daily cleanup. 7일 grace = 분쟁 시 입증.

### 6.2 Redis

```
SET rt:<hash> <value> EX 1209600         # 14d
```

→ Redis 가 자동 만료. cleanup 불필요.

**왜 Redis 의 TTL 이 편한가**
- daily batch 불필요.
- 메모리 자동 정리.

**왜 RDB cleanup 이 굳이 필요한가**
- audit 보존 + 인덱스 비대 방지.
- 단순 DELETE WHERE expires_at < ... 정도면 충분.

---

## 7. 다중 디바이스 / `/me/sessions`

**RDB**
```sql
SELECT * FROM refresh_tokens
WHERE user_id = ? AND status = 'ACTIVE'
ORDER BY issued_at DESC;
```

**Redis**
```
SMEMBERS user:<userId>:rt
foreach hash: GET rt:<hash>
```

**왜 RDB 가 더 자연**
- SELECT + WHERE + ORDER BY = 한 query.
- Redis = N+1 fetch.

---

## 8. revokeAllForUser (패스워드 변경 / 도난)

**RDB**
```sql
UPDATE refresh_tokens
SET status = 'REVOKED', revoked_at = now(), revoked_reason = ?
WHERE user_id = ? AND status = 'ACTIVE';
```

**Redis**
```
SMEMBERS user:<userId>:rt
foreach hash: DEL rt:<hash>
DEL user:<userId>:rt
```

**왜 RDB 가 더 안전**
- 트랜잭션 — 일부만 revoke 되는 경우 없음.
- audit (revoked_at / reason) 보존.

---

## 9. 비용 비교

| 항목 | RDB | Redis |
| --- | --- | --- |
| 인프라 비용 (월) | 이미 있음 | $50-500 (Elasticache) |
| 운영 부담 | DB 와 함께 | 별도 failover / backup |
| 개발 / 학습 | SQL | Redis 명령 |
| 성능 (read) | 1-5ms | 0.1-0.5ms |
| 성능 (write) | 2-10ms | 0.5-1ms |

**언제 Redis 비용이 가치가 있나**
- MAU 1000만+ — DB 부담 회피 우선.
- 글로벌 — Redis cluster 가 region 동기화 ↑.

---

## 10. 함정 모음

### 함정 1 — Redis only + AOF 없음
장애 시 토큰 손실 → 모든 user 강제 로그아웃.
→ AOF everysec + 백업.

### 함정 2 — in-memory (JVM) 저장
다중 인스턴스 동기화 X. Auto-scaling 시 즉시 깨짐.
→ 절대 X.

### 함정 3 — JWT 만 (refresh DB 없이)
즉시 무효화 X. 도난 시 14일 유효.
→ refresh 는 opaque + 서버 매핑.

### 함정 4 — RDB cleanup 없음
1년 후 1000만 row → 인덱스 비대.
→ daily cleanup.

### 함정 5 — Redis 의 SET TTL 누락
무한 누적.
→ EX 옵션 명시.

### 함정 6 — Hybrid 의 cache inconsistency
RDB rotate 했는데 Redis 의 옛 hash 가 cache hit → ROTATED 토큰 통과.
→ rotate 시 Redis 도 즉시 DEL.

### 함정 7 — Hybrid 의 read 만 Redis (write 도 dual)
write 부담 ↑ 만 발생, cache hit 활용 X.
→ Redis fail 시 RDB fallback 패턴 명시.

### 함정 8 — Redis password 평문 저장 (raw refresh)
Redis 도 DB. 평문 저장 시 보안 동일 문제.
→ token_hash 만 (SHA-256).

---

## 11. 다른 컨텍스트

### 11.1 대형 SaaS (MAU 1000만+)

```yaml
refresh-storage:
  primary: redis-cluster
  audit-log: kafka → snowflake
  reason: latency + 글로벌 분산
```

### 11.2 모놀리식 + 단일 region

```yaml
refresh-storage:
  primary: rdb-postgres
  reason: 단순성 + audit
```

### 11.3 강한 audit (금융)

```yaml
refresh-storage:
  primary: rdb-postgres
  audit-log: separate immutable log
  retention: 5years
  reason: 분쟁 / 감사 의무
```

---

## 12. 관련

- [[design-decisions|↑ hub]]
- [[token-model]] — JWT 와 refresh 의 관계
- [[../database/refresh-tokens-table]] — RDB schema 상세
- [[../security]] — 도난 대응 (rotation, reuse detection)
- [[../token-refresh-impl]] — rotation 구현
