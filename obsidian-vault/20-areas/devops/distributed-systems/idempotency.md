---
title: "Idempotency — at-least-once 의 표준 대응"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:43:00+09:00
tags: [devops, distributed-systems, idempotency]
---

# Idempotency — at-least-once 의 표준 대응

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. 정의

```
idempotent op:
  f(x) == f(f(x)) == f(f(f(x)))
  여러 번 호출해도 같은 결과.

network 의 현실:
  - request 가 timeout
  - retry → 중복 도착
  - response 가 lost → client retry
  
→ "한 번만" 보장 불가. "중복 시 안 깨짐" 만 가능.
```

---

## 2. HTTP method 의 idempotency

| | idempotent? | 의미 |
| --- | --- | --- |
| **GET** | yes | read only |
| **PUT** | yes | 같은 state 로 set |
| **DELETE** | yes | already deleted = OK |
| **HEAD** | yes | metadata |
| **OPTIONS** | yes | |
| **POST** | NO | "create new" → 중복 시 N 생성 |
| **PATCH** | depends | "amount += 5" 이면 NO |

→ POST 가 가장 위험. idempotency key 필요.

---

## 3. idempotency key 패턴 (★)

```
client → server:
  POST /orders
  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  Body: {...}

server:
  1. key 의 record check
  2. 있음 → cached response 반환 (재처리 X)
  3. 없음 → 처리 + key 저장 + response 반환
```

```java
@PostMapping("/orders")
public OrderResponse create(
        @RequestHeader("Idempotency-Key") String key,
        @RequestBody OrderRequest req) {
    
    // 이미 처리된 key?
    Optional<IdempotencyRecord> existing = repo.findById(key);
    if (existing.isPresent()) {
        return existing.get().getResponse();   // cached
    }
    
    // 처리
    OrderResponse response = orderService.create(req);
    
    // 저장
    repo.save(new IdempotencyRecord(key, response));
    
    return response;
}
```

→ Stripe / 토스 / 결제 API 의 표준.

---

## 4. idempotency 의 4 가지 처리

```
1. natural (자연스러운)
   - 같은 input → 같은 output
   - "user 의 이름 변경" → 두 번 변경 = 한 번
   
2. dedup (★ 가장 흔함)
   - request ID / idempotency key
   - 이미 처리 → cached response
   
3. compare-and-set
   - "이전 state X 인 경우만 set Y"
   - ETag / version number
   - DB unique constraint
   
4. retry-safe operation
   - 단순 op (set / delete)
```

---

## 5. DB unique constraint

```sql
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(64) UNIQUE NOT NULL,
    user_id BIGINT,
    amount DECIMAL,
    status VARCHAR(20),
    created_at TIMESTAMP
);
```

```java
try {
    paymentRepo.save(new Payment(key, userId, amount));
} catch (DuplicateKeyException e) {
    // 이미 있음 — 기존 record 가져옴
    Payment existing = paymentRepo.findByKey(key);
    return existing;
}
```

→ DB 의 atomic unique 활용. lock 없이 OK.

---

## 6. ETag (★ optimistic concurrency)

```
GET /orders/123 → ETag: "v5"

PUT /orders/123
If-Match: "v5"

server:
  current version == 5? → update + new ETag
  current version != 5? → 412 Precondition Failed
```

```java
@PutMapping("/orders/{id}")
public OrderResponse update(
        @PathVariable Long id,
        @RequestHeader("If-Match") String etag,
        @RequestBody OrderUpdate req) {
    
    Order order = repo.findById(id).orElseThrow();
    if (!order.getVersion().equals(etag)) {
        throw new PreconditionFailedException("conflict");
    }
    
    order.apply(req);
    order.incrementVersion();
    repo.save(order);
    return order;
}
```

---

## 7. messaging — at-least-once

```
Kafka / SQS:
  - at-least-once 보장 (중복 가능)
  - at-most-once (data loss 가능) 보다 안전
  - exactly-once 가능하지만 비싸

consumer side:
  1. message ID 또는 key dedup
  2. INSERT INTO processed_messages (msg_id) — unique
  3. 이미 있음 → skip
  4. 새 → process
```

```java
@KafkaListener(topics = "orders")
public void onOrder(OrderEvent event) {
    if (processedRepo.exists(event.messageId)) {
        return;   // 이미 처리. skip.
    }
    
    try {
        process(event);
        processedRepo.save(event.messageId);   // mark
    } catch (Exception e) {
        // retry (Kafka auto)
    }
}
```

→ 두 step (process + mark) 의 atomicity 가 또 issue → tx 안에 둘 다.

---

## 8. Kafka exactly-once (★)

```
Kafka 의 exactly-once semantics (EOS):
  - producer: idempotent producer + transactions
  - consumer: read_committed isolation
  - 같은 Kafka 안에서만 EOS

application code:
  - idempotency key 또는 outbox pattern
  - DB 의 unique constraint
```

```java
// Kafka producer 의 idempotent
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "my-tx-id");

producer.initTransactions();
producer.beginTransaction();
try {
    producer.send(...);
    producer.send(...);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

---

## 9. distributed retry

```
client:
  for i := 0; i < maxRetries; i++ {
      try {
          response := send(request)
          break
      } catch (TimeoutException) {
          if i == maxRetries-1: fail
          sleep(backoff(i))
      }
  }

backoff:
  exponential: 1s, 2s, 4s, 8s, ...
  + jitter (random 0-25%) — thundering herd 방지

server:
  대처:
    - idempotency key check
    - timeout 의 결과를 알 수 없음 (성공? 실패?)
    - retry 시 같은 결과 보장
```

---

## 10. tx 의 idempotency

```sql
-- ❌ 단순 update — retry 시 2번
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- ✅ idempotent — 같은 결과
UPDATE accounts 
SET balance = balance - 100, 
    transactions_processed = array_append(transactions_processed, 'tx-uuid-123')
WHERE id = 1 
  AND NOT 'tx-uuid-123' = ANY(transactions_processed);

-- 또는 별도 table
INSERT INTO processed_tx (tx_id, account_id, amount)
VALUES ('tx-uuid-123', 1, -100)
ON CONFLICT (tx_id) DO NOTHING
RETURNING tx_id;
-- RETURNING null = skip, not null = 처리
```

---

## 11. 시간 / TTL 의 idempotency

```
idempotency key 영원 저장 X.

retention:
  - 짧음 (1h): 일반 API
  - 중간 (24h): 결제
  - 긺 (30d): 큰 금액 / 법적

저장 위치:
  - Redis (TTL native)
  - DB + cleanup job
  - DynamoDB TTL
```

---

## 12. 함정

1. **POST 의 idempotency key 없음** — retry 시 중복 생성.
2. **side effect 안에서 idempotent 가정** — 외부 API 가 안 idempotent.
3. **key 너무 짧음 retention** — retry 도착 시 만료.
4. **key 만 check, response 안 cache** — 두 번째 처리 시 다른 response 가능.
5. **key + body mismatch** — 같은 key 의 다른 request → 200 인데 안 처리.
6. **inbox table 의 무한 누적** — TTL / archive.
7. **distributed retry 의 thundering herd** — jitter 추가.

---

## 13. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[saga-pattern]]
- [[messaging-patterns]]
- [[../../60-recipes/spring/product/payment/payment|↗ payment idempotency]]
