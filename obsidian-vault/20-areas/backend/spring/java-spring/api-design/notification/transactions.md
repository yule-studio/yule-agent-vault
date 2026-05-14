---
title: "notification transactions — race + AFTER_COMMIT 패턴"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - transactions
---

# notification transactions — race + AFTER_COMMIT 패턴

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[notification|↑ hub]]**

---

## 1. 격리 수준

| 영역 | 격리 | 이유 |
| --- | --- | --- |
| outbox INSERT | REQUIRED (같은 도메인 tx) | 도메인 commit 와 함께 |
| Worker pickup | REQUIRES_NEW + SKIP LOCKED | 락 시간 짧게 |
| 외부 채널 호출 | **트랜잭션 밖** | DB 락 회피 |
| device 등록 / 갱신 | READ_COMMITTED + UPSERT | 단순 |

---

## 2. 흐름 — 4단계 트랜잭션

### 2.1 Outbox INSERT (도메인 tx 안)

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onOrderPaid(OrderPaid ev) {
    // 이 시점에 order tx 는 이미 commit 됨
    // 새 tx 시작 (REQUIRES_NEW 또는 default)
    outboxService.enqueue(...);
}
```

→ AFTER_COMMIT 이라 별도 tx — order rollback 시 알림 X.

### 2.2 Worker pickup (짧은 tx)

```java
@Transactional
public List<NotificationOutboxRow> pickup(int limit) {
    var rows = repo.findPendingForUpdateSkipLocked(limit);  // SELECT FOR UPDATE SKIP LOCKED
    rows.forEach(r -> r.markProcessing(now()));
    repo.saveAll(rows);
    return rows;
}
```

### 2.3 외부 호출 (트랜잭션 밖)

```java
// pickup() 이 반환한 rows 를 트랜잭션 밖에서 처리
public void process(List<NotificationOutboxRow> rows) {
    for (var row : rows) {
        try {
            var result = router.route(row);   // 외부 FCM/APNs/...
            updateResult(row, result);
        } catch (Exception e) {
            scheduleRetry(row, e);
        }
    }
}
```

### 2.4 결과 기록 (짧은 tx)

```java
@Transactional
public void updateResult(NotificationOutboxRow row, DeliveryResult result) {
    if (result.isSuccess()) row.markSent(now());
    else if (result.isPermanent()) row.markFailed(result.errorMessage());
    else row.recordTransientFailure(...);
    repo.save(row);
}
```

---

## 3. Race 시나리오

### 3.1 동일 row 중복 처리

| 시나리오 | 원인 | 방어 |
| --- | --- | --- |
| Worker 2 instance 동시 SELECT | scheduler race | SKIP LOCKED + ShedLock |
| 옛 row 가 stuck (PROCESSING 30분+) | worker crash | reaper cron — PROCESSING 더 1시간 → PENDING reset |

### 3.2 dedup race

```java
// event_id UNIQUE — 같은 event 2 listener call 시 두 번째 INSERT 가 fail
try {
    outbox.save(new NotificationOutboxRow(event_id, ...));
} catch (DataIntegrityViolationException e) {
    log.info("dedup skip: {}", event_id);
    return;
}
```

### 3.3 device 등록 race

```java
// UPSERT pattern (PostgreSQL ON CONFLICT)
INSERT INTO user_devices (user_id, token, platform, ...)
VALUES (?, ?, ?, ...)
ON CONFLICT (token) DO UPDATE
SET user_id = EXCLUDED.user_id,           -- 다른 user 가 같은 token (device 양도)
    updated_at = now(),
    active = true;
```

---

## 4. AFTER_COMMIT 함정

### 함정 1 — 동기 listener (트랜잭션 안)

```java
// X — order tx 안에서 FCM 호출 시 DB 락 + cascade
@EventListener
public void on(OrderPaid ev) {
    fcm.send(...);   // 외부 호출
}

// O — AFTER_COMMIT
@TransactionalEventListener(phase = AFTER_COMMIT)
public void on(OrderPaid ev) {
    outbox.enqueue(...);   // DB INSERT only
}
```

### 함정 2 — AFTER_COMMIT 시 새 tx 명시

```java
// outbox.enqueue 가 자동으로 새 tx 시작 (default propagation REQUIRED)
// AFTER_COMMIT 후 활성 tx 없음 → @Transactional 필요
@Service
public class NotificationOutboxService {
    @Transactional
    public void enqueue(...) { ... }    // 새 tx
}
```

---

## 5. Stuck row reaper

```java
@Scheduled(cron = "0 */5 * * * *")    // 5분
public void reapStuck() {
    var stuck = repo.findStuck(Duration.ofHours(1));
    for (var row : stuck) {
        if (row.attempts() < maxAttempts)
            row.markPending(now());   // retry
        else
            row.markDlq("stuck > 1h");
        repo.save(row);
    }
}
```

→ worker crash 시 PROCESSING 상태로 영구 잠김 방지.

---

## 6. 관련

- [[notification|↑ hub]]
- [[design-decisions/outbox-pattern]]
- [[implementation/outbox-worker-impl]]
- [[pitfalls/outbox-pitfalls]]
- [[../signup/transactions|↗ signup transactions]]
