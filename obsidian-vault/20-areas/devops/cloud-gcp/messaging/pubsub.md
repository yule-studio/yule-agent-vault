---
title: "GCP Pub/Sub"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:50:00+09:00
tags:
  - gcp
  - messaging
  - pubsub
---

# GCP Pub/Sub

**[[messaging|↑ Messaging]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

**글로벌** message broker — queue + pub/sub 통합. AWS SQS + SNS + Kinesis 의 강력 통합.

---

## 2. 구조

```
Publisher → Topic ─┬→ Subscription 1 → Subscriber 1 (pull / push / BigQuery / Cloud Storage)
                   └→ Subscription 2 → Subscriber 2
```

- 1 topic → N subscription
- 각 subscription = 독립적 (queue 비슷)
- 글로벌 (region 무관)
- at-least-once
- 7일 retention (옵션 31일)

---

## 3. 사용

```bash
gcloud pubsub topics create myapp-events
gcloud pubsub subscriptions create email-worker \
  --topic=myapp-events --ack-deadline=60

# publish
gcloud pubsub topics publish myapp-events --message='{"event":"signup"}'

# pull
gcloud pubsub subscriptions pull email-worker --auto-ack
```

```python
from google.cloud import pubsub_v1
publisher = pubsub_v1.PublisherClient()
topic = publisher.topic_path("PROJECT", "myapp-events")
publisher.publish(topic, b'{"event":"signup"}', user_id="alice")

subscriber = pubsub_v1.SubscriberClient()
def callback(msg):
    print(msg.data)
    msg.ack()
subscriber.subscribe(subscriber.subscription_path("PROJECT","email-worker"), callback)
```

---

## 4. Subscription type

| Type | 의미 |
| --- | --- |
| **Pull** | subscriber 가 polling |
| **Push** | Pub/Sub 가 HTTPS POST |
| **BigQuery** | 자동 BigQuery insert |
| **Cloud Storage** | 자동 GCS file 작성 |

→ Cloud Run + Push subscription = serverless event 처리.

---

## 5. Filter / Ordering

```hcl
filter = "attributes.event_type = \"signup\""
enable_message_ordering = true                  # 같은 ordering_key 순서
```

---

## 6. DLQ

```hcl
resource "google_pubsub_subscription" "main" {
  dead_letter_policy {
    dead_letter_topic     = google_pubsub_topic.dlq.id
    max_delivery_attempts = 5
  }
}
```

---

## 7. 비용

```
$40 / TB throughput
storage: $0.27 / GB·월
DLQ / cross-region snapshots 추가

Free tier: 10 GB / month
```

---

## 8. 함정

- ack deadline 너무 짧음 = 중복
- push subscription endpoint = HTTPS + auth (Cloud Run service account)
- ordering 활성 시 throughput ↓
- BigQuery subscription = schema 매칭 필요
- 7 일 retention default — replay 필요시 31일

---

## 9. 관련

- [[messaging]]
- [[../compute/cloud-run]] — push subscriber
- [[../../../computer-science/network/rpc-messaging/mqtt-amqp-kafka|↗ MQ 이론]]
