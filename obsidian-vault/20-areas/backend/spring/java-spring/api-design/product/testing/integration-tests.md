---
title: "Integration tests — Testcontainers + EmbeddedKafka"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:56:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - testing
  - integration
---

# Integration tests — Testcontainers + EmbeddedKafka

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[testing|↑ hub]]**

---

## 1. Base config

```java
@SpringBootTest
@Testcontainers
@AutoConfigureMockMvc
abstract class IntegrationTestBase {

    @Container static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    @Container static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        r.add("spring.datasource.url", postgres::getJdbcUrl);
        r.add("spring.datasource.username", postgres::getUsername);
        r.add("spring.datasource.password", postgres::getPassword);
        r.add("spring.redis.host", redis::getHost);
        r.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }
}
```

---

## 2. 결제 confirm (Toss sandbox)

```java
class PaymentConfirmIT extends IntegrationTestBase {

    @Autowired MockMvc mvc;
    @MockBean PaymentGateway tossGateway;

    @Test void confirm_success() throws Exception {
        var order = createOrder(50000);
        when(tossGateway.confirm(any())).thenReturn(PgConfirmResult.success(...));

        mvc.perform(post("/api/v1/payments/confirm")
                .header("Idempotency-Key", "key-1")
                .contentType(JSON)
                .content("""
                    { "paymentKey": "pk_123", "orderId": "%s", "amount": 50000 }
                    """.formatted(order.orderNumber())))
            .andExpect(status().isOk());

        // verify
        assertThat(payments.findByOrderId(order.id()).orElseThrow().status())
            .isEqualTo(PaymentStatus.DONE);
    }

    @Test void duplicate_idempotencyKey_same_response() throws Exception {
        // 첫 호출
        mvc.perform(post(...).header("Idempotency-Key", "key-1")...)
            .andExpect(status().isOk());

        // 두번째 — PG 호출 X (cache 사용)
        mvc.perform(post(...).header("Idempotency-Key", "key-1")...)
            .andExpect(status().isOk());

        verify(tossGateway, times(1)).confirm(any());
    }

    @Test void amount_tampered_rejected() throws Exception {
        var order = createOrder(50000);
        mvc.perform(post("/api/v1/payments/confirm")
                .header("Idempotency-Key", "key-x")
                .contentType(JSON)
                .content("""
                    { "paymentKey": "pk", "orderId": "%s", "amount": 100 }
                    """.formatted(order.orderNumber())))
            .andExpect(status().is4xxClientError());
    }
}
```

---

## 3. 재고 동시성

```java
@Test void 100_threads_1_stock_only_one_succeeds() throws Exception {
    var sku = createSkuWithStock(1);
    var executor = Executors.newFixedThreadPool(100);
    var success = new AtomicInteger();
    var latch = new CountDownLatch(100);

    for (int i = 0; i < 100; i++) {
        executor.submit(() -> {
            try {
                inventory.decrement(sku.id(), 1);
                success.incrementAndGet();
            } catch (InsufficientStockException ignored) {}
            finally { latch.countDown(); }
        });
    }
    latch.await();

    assertThat(success.get()).isEqualTo(1);
}
```

---

## 4. 디지털 워터마크 worker (Awaitility)

```java
@Test void payment_approved_triggers_watermark() throws Exception {
    var bookProduct = createBookProduct();
    var order = createPaidOrder(bookProduct);

    Awaitility.await().atMost(Duration.ofSeconds(30))
        .untilAsserted(() -> {
            var d = deliveries.findByOrderId(order.id()).get(0);
            assertThat(d.status()).isEqualTo(DeliveryStatus.READY);
            assertThat(d.storageFileId()).isNotBlank();
        });
}
```

---

## 5. Kafka (EmbeddedKafka, F10+)

```java
@SpringBootTest
@EmbeddedKafka(partitions = 3, topics = "product.payment.events.v1")
class PaymentKafkaIT {

    @Autowired EmbeddedKafkaBroker broker;
    @Autowired PaymentConfirmService service;

    @Test void payment_approved_publishes_to_kafka() throws Exception {
        var consumer = createConsumer(broker, "test-group");
        consumer.subscribe(List.of("product.payment.events.v1"));

        service.confirm("key", "pk", orderId, Money.krw(50000), userId);

        Awaitility.await().untilAsserted(() -> {
            var records = consumer.poll(Duration.ofMillis(500));
            assertThat(records.count()).isEqualTo(1);
            assertThat(records.iterator().next().key()).isEqualTo(orderId.value());
        });
    }
}
```

자세히: [[../design-decisions/kafka-event-driven]] · [[../implementation/kafka-integration]].

---

## 6. 관련

- [[testing|↑ hub]]
- [[unit-tests]]
- [[pg-mock-tests]]
