---
title: "Observability — 메트릭 / Prometheus / Grafana"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:04:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - operations
  - observability
---

# Observability — 메트릭 / Prometheus / Grafana

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[operations|↑ hub]]**

---

## 1. 메트릭

| Metric | type | tag |
| --- | --- | --- |
| `product_order_created_total` | counter | type |
| `product_payment_attempt_total` | counter | provider, status |
| `product_payment_confirm_duration_seconds` | histogram | provider |
| `product_payment_amount_krw_total` | counter | provider, status |
| `product_refund_amount_krw_total` | counter | reason |
| `product_inventory_low_total` | counter | sku_id |
| `product_digital_delivery_duration_seconds` | histogram | — |
| `product_digital_delivery_attempt_total` | counter | status |
| `product_webhook_received_total` | counter | provider, event_type |
| `product_kafka_outbox_pending_count` | gauge | — |
| `product_kafka_consumer_lag` | gauge | topic, group |

---

## 2. Spring Boot 통합

```yaml
management:
  endpoints.web.exposure.include: prometheus,health,info
  metrics.tags.application: product
  metrics.distribution:
    percentiles[product_payment_confirm_duration_seconds]: 0.5, 0.95, 0.99
    sla[product_payment_confirm_duration_seconds]: 500ms, 1s, 2s
```

```java
@Service
class PaymentConfirmService {
    private final Timer confirmTimer;

    public PaymentConfirmService(MeterRegistry reg) {
        this.confirmTimer = Timer.builder("product_payment_confirm_duration_seconds")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(reg);
    }

    public PaymentResult confirm(...) {
        return confirmTimer.recordCallable(() -> {
            // ...
        });
    }
}
```

---

## 3. Grafana 대시보드

| Panel | Query |
| --- | --- |
| 결제 TPS | `rate(product_payment_attempt_total[1m])` |
| 결제 p95 latency | `histogram_quantile(0.95, ...)` |
| 매출 (KRW) | `sum(rate(product_payment_amount_krw_total{status="DONE"}[5m]))` |
| Kafka lag (per consumer) | `product_kafka_consumer_lag` |
| Outbox pending | `product_kafka_outbox_pending_count` |
| webhook 5xx | `rate(product_webhook_received_total{status="5xx"}[5m])` |

---

## 4. 알람

| 조건 | 채널 |
| --- | --- |
| 결제 5xx > 1% | #payment-alerts (P0) |
| 결제 p95 > 2s | #payment-alerts (P1) |
| outbox pending > 1000 | #infra (P1) |
| Kafka lag > 10000 | #infra (P0) |
| amount mismatch (대사) | #payment-alerts (P0) |
| 자동 모더 spike | #fraud |

---

## 5. 관련

- [[operations|↑ hub]]
- [[runbook]]
- [[reconciliation]]
