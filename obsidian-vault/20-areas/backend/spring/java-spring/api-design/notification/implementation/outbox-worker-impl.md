---
title: "Outbox Worker 구현 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:42:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, outbox, worker]
---

# Outbox Worker 구현 ★

**[[implementation|↑ hub]]**

---

## 1. 코드

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxPublishWorker {

    private final NotificationOutboxRepository repo;
    private final ChannelRouter router;
    private final RetryPolicy retryPolicy;
    private final NotificationDlqRepository dlqRepo;
    private final SlackClient slack;
    private final Clock clock;
    private final OutboxProperties props;

    @Scheduled(fixedDelay = 1_000)    // 1s
    @SchedulerLock(name = "notifOutboxWorker",
                   lockAtMostFor = "5m",
                   lockAtLeastFor = "1s")
    public void run() {
        try {
            var rows = pickup(props.batchSize());
            for (var row : rows) processOne(row);
        } catch (Exception e) {
            log.error("outbox worker outer fail", e);
        }
    }

    @Transactional
    public List<Notification> pickup(int limit) {
        var rows = repo.findPendingForUpdateSkipLocked(limit, clock.now());
        rows.forEach(r -> r.markProcessing(clock.now()));
        repo.saveAll(rows);
        return rows;
    }

    public void processOne(Notification n) {
        // 트랜잭션 밖 (외부 호출)
        try {
            var results = router.route(n);
            updateSuccess(n, results);
        } catch (TransientException e) {
            scheduleRetry(n, e.getMessage());
        } catch (PermanentException e) {
            moveToDlq(n, e.getMessage(), "permanent");
        } catch (Exception e) {
            log.error("unknown failure: {}", n.id(), e);
            scheduleRetry(n, e.getMessage());
        }
    }

    @Transactional
    public void updateSuccess(Notification n, Map<ChannelType, DeliveryResult> results) {
        // permanent fail 있으면 부분 실패 — 일부 retry
        var allSuccess = results.values().stream().allMatch(DeliveryResult::success);
        var anyPermanent = results.values().stream().anyMatch(DeliveryResult::permanent);

        if (allSuccess) {
            n.markSent(results, clock.now());
        } else if (anyPermanent && results.size() == 1) {
            moveToDlq(n, "permanent channel failure", "channel-permanent");
        } else {
            scheduleRetry(n, "partial channel failure");
        }
        repo.save(n);
    }

    @Transactional
    public void scheduleRetry(Notification n, String error) {
        var next = retryPolicy.nextAttempt(n.attempts(), clock.now());
        if (next == null || n.attempts() >= n.maxAttempts()) {
            moveToDlq(n, error, "max-attempts");
        } else {
            n.recordTransientFailure(error, next);
            repo.save(n);
        }
    }

    @Transactional
    public void moveToDlq(Notification n, String error, String category) {
        n.moveToDlq(error, clock.now());
        repo.save(n);
        dlqRepo.save(NotificationDlqRow.from(n, error));
        slack.alert("notification dlq", Map.of(
            "id", n.id(),
            "userId", n.userId(),
            "type", n.type(),
            "category", category,
            "error", error));
    }

    @Scheduled(cron = "0 */5 * * * *")    // 5분
    public void reapStuck() {
        var stuck = repo.findStuck(Duration.ofHours(1));
        for (var n : stuck) {
            log.warn("stuck reset: {}", n.id());
            n.markPending(clock.now());
            repo.save(n);
        }
    }

    @Scheduled(cron = "0 0 4 * * *")    // 매일 04:00
    public void cleanup() {
        int deleted = repo.deactivateOldSent(Duration.ofDays(30));
        log.info("cleaned {} old SENT rows", deleted);
    }
}
```

---

## 2. 함정

- pickup 의 tx 가 외부 호출 까지 → DB 락 30s.
- 처리 중 worker crash → PROCESSING 영구 → reapStuck cron.
- ShedLock 없이 multi-instance → 같은 batch 중복.

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/outbox-pattern]]
- [[../design-decisions/retry-strategy]]
- [[../transactions]]
