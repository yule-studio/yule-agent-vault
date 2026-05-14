---
title: "DLQ replay — admin 수동 처리"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:22:00+09:00
tags: [backend, java-spring, api-design, notification, operations, dlq]
---

# DLQ replay — admin 수동 처리 ★

**[[operations|↑ hub]]**

---

## 1. UI / Admin endpoint

```http
GET    /api/v1/admin/notifications/dlq?cursor=...&type=...
POST   /api/v1/admin/notifications/dlq/{id}/replay
POST   /api/v1/admin/notifications/dlq/{id}/resolve
POST   /api/v1/admin/notifications/dlq/batch-replay   # 다중
GET    /api/v1/admin/notifications/dlq/stats           # 카테고리별 통계
```

---

## 2. 코드

```java
@Service
@RequiredArgsConstructor
public class DlqReplayService {

    private final NotificationDlqRepository dlqRepo;
    private final NotificationOutboxRepository outboxRepo;
    private final Clock clock;

    @Transactional
    public void replay(String dlqId, UserId admin) {
        var dlq = dlqRepo.findById(dlqId).orElseThrow();
        if (dlq.isResolved()) throw new AlreadyResolvedException();

        // 새 outbox row 생성 (event_id 재발급)
        var newRow = Notification.create(
            NotificationId.next(),
            "replay-" + dlq.eventId(),     // 새 event_id (UNIQUE 충돌 방지)
            dlq.type(),
            dlq.userId(),
            dlq.templateKey(),
            dlq.variables(),
            dlq.channels(),
            NotificationPriority.HIGH,     // replay 우선
            clock.now());
        outboxRepo.save(newRow);

        dlq.markReplayed(admin, clock.now());
        dlqRepo.save(dlq);
    }

    @Transactional
    public void resolve(String dlqId, UserId admin, String resolution) {
        var dlq = dlqRepo.findById(dlqId).orElseThrow();
        dlq.markResolved(admin, resolution, clock.now());
        dlqRepo.save(dlq);
    }

    public DlqStats stats(Duration window) {
        return dlqRepo.statsByReason(window);
    }
}
```

---

## 3. 자동 처리 패턴

```java
@Scheduled(cron = "0 0 * * * *")    // hourly
public void autoResolveExpected() {
    // 자동 처리 가능한 패턴
    var unregistered = dlqRepo.findByReasonContaining("UNREGISTERED");
    for (var d : unregistered) {
        // device 이미 비활성화됨 → DLQ resolve
        d.markResolved("SYSTEM", "device-unregistered-auto", clock.now());
        dlqRepo.save(d);
    }
}
```

---

## 4. 관련

- [[operations|↑ hub]]
- [[runbook]]
- [[../database/notification-dlq-table]]
- [[../design-decisions/retry-strategy]]
