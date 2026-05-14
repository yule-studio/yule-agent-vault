---
title: "Azure Service Bus"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:50:00+09:00
tags:
  - azure
  - messaging
  - service-bus
---

# Azure Service Bus

**[[messaging|↑ Messaging]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

엔터프라이즈 message broker — Queue + Topic (Pub/Sub) + AMQP 1.0 + 트랜잭션. AWS SQS + SNS + 더 풍부 (옛 enterprise 친화).

---

## 2. 두 모드

| | Queue | Topic |
| --- | --- | --- |
| 패턴 | 1:1 | Pub/Sub (1:N) |
| Subscription | X | N 개 |
| Filtering | X | SQL filter |

---

## 3. Tier

| | Basic | Standard | Premium |
| --- | --- | --- | --- |
| Topic | X | ✅ | ✅ |
| Dedicated 자원 | X | X | ✅ |
| VNet | X | X | ✅ |
| 가격 | 매우 쌈 | low | 비쌈 |

→ 운영 = Standard 또는 Premium.

---

## 4. 사용

```bash
az servicebus namespace create -g myapp -n myapp-sb -l koreacentral --sku Standard

az servicebus queue create -g myapp --namespace-name myapp-sb -n jobs \
  --max-delivery-count 5 --enable-dead-lettering-on-message-expiration true

az servicebus topic create -g myapp --namespace-name myapp-sb -n events
az servicebus topic subscription create -g myapp --namespace-name myapp-sb \
  --topic-name events --name email-worker
```

```hcl
resource "azurerm_servicebus_namespace" "sb" {
  name                = "myapp-sb"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "Standard"
}

resource "azurerm_servicebus_queue" "jobs" {
  name                                 = "jobs"
  namespace_id                         = azurerm_servicebus_namespace.sb.id
  max_delivery_count                   = 5
  dead_lettering_on_message_expiration = true
  lock_duration                        = "PT1M"
}
```

---

## 5. SDK

```python
from azure.servicebus import ServiceBusClient, ServiceBusMessage

client = ServiceBusClient.from_connection_string(conn_str)

with client.get_queue_sender("jobs") as sender:
    sender.send_messages(ServiceBusMessage('{"task":"send_email"}'))

with client.get_queue_receiver("jobs", max_wait_time=20) as receiver:
    for msg in receiver:
        try:
            handle(msg)
            receiver.complete_message(msg)
        except Exception:
            receiver.abandon_message(msg)        # 또는 dead_letter
```

---

## 6. 기능

- **Transaction** — 여러 메시지 atomic
- **Sessions** — FIFO + ordering per session
- **Scheduling** — 미래 시각 전달
- **DLQ** — 자동
- **Auto-forwarding** — queue / subscription 간
- **Duplicate detection**
- **Geo-disaster recovery** (Premium)

---

## 7. 비용 (Standard)

```
Namespace: $0.0135 / hour = ~$10/월
Operations: $0.05 / 1M
```

(Premium 은 messaging unit 단위, 비쌈).

---

## 8. 함정

- Topic = Standard 부터
- Lock duration — 처리 시간보다 길게
- max_delivery_count 후 DLQ
- entity name = case-sensitive
- AMQP / SDK 버전

---

## 9. 관련

- [[messaging]]
- [[event-grid]]
- [[../../../computer-science/network/rpc-messaging/mqtt-amqp-kafka|↗ MQ 이론]]
