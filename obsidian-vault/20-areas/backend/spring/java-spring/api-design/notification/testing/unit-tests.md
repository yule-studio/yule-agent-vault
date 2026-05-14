---
title: "Unit tests"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:10:00+09:00
tags: [backend, java-spring, api-design, notification, testing, unit]
---

# Unit tests

**[[testing|↑ hub]]**

---

## Notification aggregate

```java
@Test void markProcessing_PENDING_only() {
    var n = Notification.create(...);
    n.markProcessing(now);
    assertThat(n.status()).isEqualTo(NotificationStatus.PROCESSING);
    assertThat(n.attempts()).isEqualTo(1);
}

@Test void markSent_emits_event() {
    var n = processingNotification();
    n.markSent(Map.of(FCM, success()), now);
    assertThat(n.events()).extracting("class.simpleName")
        .contains("NotificationSent");
}

@Test void moveToDlq_after_max() {
    var n = retriedNotification(5);  // attempts=5
    n.moveToDlq("max", now);
    assertThat(n.status()).isEqualTo(NotificationStatus.DLQ);
}
```

## TemplateRenderer

```java
@Test void substitute_replaces_variables() {
    var template = "안녕 {{userName}}님";
    var result = renderer.substitute(template, Map.of("userName", "홍길동"), FCM);
    assertThat(result).isEqualTo("안녕 홍길동님");
}

@Test void escape_html_for_email() {
    var template = "Hello {{name}}";
    var result = renderer.substitute(template, Map.of("name", "<script>"), EMAIL);
    assertThat(result).contains("&lt;script&gt;");
}
```

## RetryPolicy

```java
@Test void exp_backoff_increases() {
    var p = new RetryPolicy();
    var now = Instant.parse("2026-05-14T00:00:00Z");
    assertThat(p.nextAttempt(0, now)).isEqualTo(now.plusSeconds(30));
    assertThat(p.nextAttempt(1, now)).isEqualTo(now.plusSeconds(120));
}

@Test void null_after_max() {
    var p = new RetryPolicy();
    assertThat(p.nextAttempt(5, now)).isNull();
}
```

---

## 관련

- [[testing|↑ hub]]
- [[../domain-model/notification-aggregate]]
