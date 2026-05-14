---
title: "OCI Streaming"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:35:00+09:00
tags:
  - oci
  - messaging
  - streaming
  - kafka
---

# OCI Streaming

**[[messaging|↑ Messaging]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

매니지드 streaming — **Kafka-compatible** (Kafka API). Kinesis / Pub/Sub / Event Hubs 동등.

---

## 2. 사용

```hcl
resource "oci_streaming_stream_pool" "pool" {
  compartment_id = var.compartment_ocid
  name           = "myapp-pool"
  kafka_settings {
    auto_create_topics_enable = false
    log_retention_hours       = 24
  }
}

resource "oci_streaming_stream" "events" {
  compartment_id = var.compartment_ocid
  name           = "events"
  partitions     = 3
  stream_pool_id = oci_streaming_stream_pool.pool.id
  retention_in_hours = 24
}
```

---

## 3. Kafka API 로 사용

```bash
# auth = SASL/PLAIN with auth token
KAFKA_BOOTSTRAP="cell-1.streaming.ap-seoul-1.oci.oraclecloud.com:9092"
USERNAME="<tenancy>/<username>/<stream-pool-id>"
PASSWORD="<auth-token>"
```

```python
from confluent_kafka import Producer, Consumer

p = Producer({
    "bootstrap.servers": KAFKA_BOOTSTRAP,
    "security.protocol": "SASL_SSL",
    "sasl.mechanism": "PLAIN",
    "sasl.username": USERNAME,
    "sasl.password": PASSWORD,
})
p.produce("events", value=b'{"k":"v"}')
p.flush()
```

→ Kafka 의 모든 코드 / 도구 재사용 (Kafka Connect / Streams / KSQL — 일부 제약).

---

## 4. OCI 자체 API

`PutMessages` / `GetMessages` REST API — Kafka 와 별도.

---

## 5. Stream Pool

여러 stream 의 컨테이너 — 비용 효율 / Kafka cluster 추상.

---

## 6. 가격

```
$0.030 / 시간 / partition
+ $0.025 / GB write
+ $0.025 / GB read
```

→ Kinesis 와 비슷한 가격.

---

## 7. 함정

- partition 늘리기 만 가능 (줄이기 X)
- Kafka Connect = OCI 안 자체 (또는 외부 cluster)
- retention 최대 168 시간 (7 일)
- consumer offset = Kafka 자체 또는 별도 저장
- 응용 reconnect 의 SASL token 갱신

---

## 8. 관련

- [[messaging]]
- [[notifications]]
- [[../../../database/kafka/kafka|↗ Kafka]]
