---
title: "Notification Aggregate (outbox row)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:04:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - domain-model
  - aggregate
---

# Notification Aggregate (outbox row)

**[[domain-model|↑ hub]]**

---

## 1. Java

```java
public final class Notification {
    private final NotificationId id;
    private final String eventId;
    private final NotificationType type;
    private final UserId userId;
    private final TemplateKey templateKey;
    private final Map<String, Object> variables;
    private final List<ChannelType> channels;
    private final Priority priority;
    private final boolean aggregated;
    private NotificationStatus status;
    private int attempts;
    private final int maxAttempts;
    private String lastError;
    private Map<ChannelType, DeliveryResult> results;
    private Instant scheduledAt;
    private Instant nextAttemptAt;
    private Instant sentAt;
    private Instant failedAt;
    private final List<DomainEvent> events = new ArrayList<>();

    public static Notification create(NotificationId id, String eventId,
                                       NotificationType type, UserId userId,
                                       TemplateKey templateKey,
                                       Map<String, Object> variables,
                                       List<ChannelType> channels,
                                       Priority priority,
                                       Instant scheduledAt) {
        return new Notification(id, eventId, type, userId, templateKey, variables,
                                channels, priority, /* aggregated */ false,
                                NotificationStatus.PENDING, scheduledAt);
    }

    public void markProcessing(Instant now) {
        require(status == NotificationStatus.PENDING);
        this.status = NotificationStatus.PROCESSING;
        this.attempts++;
    }

    public void markSent(Map<ChannelType, DeliveryResult> results, Instant now) {
        this.status = NotificationStatus.SENT;
        this.results = results;
        this.sentAt = now;
        events.add(new NotificationSent(id, userId, type, channels, now));
    }

    public void recordTransientFailure(String error, Instant nextAttempt) {
        this.lastError = error;
        this.nextAttemptAt = nextAttempt;
        this.status = NotificationStatus.PENDING;
    }

    public void moveToDlq(String reason, Instant now) {
        this.status = NotificationStatus.DLQ;
        this.failedAt = now;
        this.lastError = reason;
        events.add(new NotificationDlq(id, userId, type, reason, attempts, now));
    }

    public boolean shouldDelay(Instant now) {
        return scheduledAt.isAfter(now);   // quiet hours
    }
}
```

---

## 2. 불변식

- PENDING → PROCESSING → SENT (또는 PENDING retry → DLQ).
- attempts ≤ maxAttempts.
- DLQ 후 상태 변경 X (admin replay 만 새 row).

---

## 3. 이벤트

- NotificationCreated
- NotificationSent
- NotificationFailed
- NotificationDlq

---

## 4. 관련

- [[domain-model|↑ hub]]
- [[../database/notification-outbox-table]]
- [[../enums/notification-status]]
