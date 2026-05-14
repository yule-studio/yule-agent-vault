---
title: "Integration tests"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:12:00+09:00
tags: [backend, java-spring, api-design, notification, testing, integration]
---

# Integration tests

**[[testing|↑ hub]]**

---

## Base

```java
@SpringBootTest
@Testcontainers
abstract class IntegrationTestBase {
    @Container static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    @Container static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);
}
```

---

## Outbox listener (도메인 이벤트 → outbox)

```java
@Test void payment_approved_inserts_outbox() {
    publisher.publishEvent(new PaymentApproved(...));

    await().untilAsserted(() -> {
        var rows = outbox.findByUserId(buyerId);
        assertThat(rows).hasSize(1);
        assertThat(rows.get(0).type()).isEqualTo("PAYMENT_APPROVED");
    });
}

@Test void rollback_skips_outbox() {
    assertThatThrownBy(() -> service.simulateRollback(...)).isInstanceOf(...);

    var rows = outbox.findByUserId(userId);
    assertThat(rows).isEmpty();    // AFTER_COMMIT 안 호출
}
```

---

## Worker concurrent (SKIP LOCKED)

```java
@Test void multi_worker_no_duplicate() throws Exception {
    insertPending(100);

    var executor = Executors.newFixedThreadPool(5);
    for (int i = 0; i < 5; i++) {
        executor.submit(() -> worker.run());
    }
    executor.shutdown();
    executor.awaitTermination(30, SECONDS);

    var sentCount = outbox.countByStatus("SENT");
    assertThat(sentCount).isEqualTo(100);    // 중복 X
}
```

---

## FCM (WireMock)

```java
@AutoConfigureWireMock(port = 0)
class FcmChannelIT extends IntegrationTestBase {

    @Test void success() {
        stubFor(post(urlEqualTo("/fcm/send"))
            .willReturn(okJson("""
                { "name": "projects/x/messages/abc123" }
                """)));

        var result = channel.send(payload(), device());
        assertThat(result.success()).isTrue();
    }

    @Test void unregistered_deactivates() {
        stubFor(post(urlEqualTo("/fcm/send"))
            .willReturn(jsonResponse("""
                { "error": { "details": [{ "errorCode": "UNREGISTERED" }] } }
                """, 404)));

        var result = channel.send(payload(), device());
        assertThat(result.permanent()).isTrue();
        assertThat(deviceRepo.findById(device.id()).active()).isFalse();
    }
}
```

---

## 관련

- [[testing|↑ hub]]
- [[unit-tests]]
- [[channel-mock-tests]]
