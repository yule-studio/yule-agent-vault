---
title: "MQTT / AMQP / Kafka — 메시지 큐"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T08:25:00+09:00
tags:
  - network
  - messaging
  - mqtt
  - amqp
  - kafka
---

# MQTT / AMQP / Kafka — 메시지 큐

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | IoT / Enterprise / Big data |

**[[rpc-messaging|↑ RPC/Messaging]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | MQTT | AMQP | Kafka |
| --- | --- | --- | --- |
| **표준** | OASIS | ISO/IEC 19464 | de facto |
| **포트** | 1883/8883 (TLS) | 5672/5671 (TLS) | 9092 |
| **용도** | IoT | Enterprise | Big data / log |
| **시작** | 1999 (IBM) | 2003 (JPMC) | 2010 (LinkedIn) |

---

## 1. 한 줄 정의

서비스 간 **비동기 메시징** 의 3 대 표준:
- **MQTT** — IoT 의 가벼운 pub-sub
- **AMQP** — 엔터프라이즈의 강력한 큐
- **Kafka** — 빅데이터 / 스트림의 분산 log

---

## 2. MQTT — Message Queuing Telemetry Transport

### 정의
- 1999 IBM (Andy Stanford-Clark / Arlen Nipper)
- 매우 가벼운 pub-sub
- IoT / 임베디드

### 특징
- Header 2 byte (최소)
- 저전력 / 저대역 친화
- Pub-Sub 모델

### Topic
```
sensor/room1/temperature
sensor/room1/+              ← wildcard (한 level)
sensor/#                    ← wildcard (모든 level)
```

### QoS
| QoS | 의미 |
| --- | --- |
| 0 | At most once (fire and forget) |
| 1 | At least once (ack) |
| 2 | Exactly once (2-phase handshake) |

### Retained message
- Topic 별 마지막 메시지 저장
- 새 구독자 — 즉시 받음

### Last Will
- 클라이언트 비정상 종료 시 — server 가 broadcast
- 디바이스 offline 알림

### MQTT-SN (Sensor Networks)
- UDP / ZigBee 위
- 더 작은 헤더

### MQTT 5.0 (2018+)
- Reason code
- Shared subscription (load balancing)
- Topic alias (반복 topic 단축)

### Broker
- **Mosquitto** (open source)
- **EMQX** (cluster)
- **HiveMQ** (commercial)
- **AWS IoT Core**

---

## 3. AMQP — Advanced Message Queuing Protocol

### 정의
- 2003 JPMorgan
- ISO/IEC 19464 (2014)
- 엔터프라이즈 — 신뢰 / 트랜잭션

### 모델 — Exchange / Queue / Binding

```
Publisher → Exchange → Binding → Queue → Consumer
```

### Exchange 유형
| | 라우팅 |
| --- | --- |
| **Direct** | routing_key 정확 매치 |
| **Fanout** | 모든 queue 로 |
| **Topic** | wildcard (`*.error`, `app.#`) |
| **Headers** | header 매치 |

### Queue
- FIFO
- Durable (재시작 후 유지)
- Exclusive (한 connection 만)
- Auto-delete

### 흐름
```
Publisher: amqp.publish(exchange='log', key='app.error', body='...')
Exchange (topic): 'app.error' → Queue 'errors'
Consumer: queue 'errors' → 메시지 처리 → ack
```

### Ack
- Manual ack — consumer 가 처리 후 ack
- ack 안 오면 — 다른 consumer 재전달

### Broker
- **RabbitMQ** (가장 흔함)
- **Apache Qpid**
- **ActiveMQ**
- **Solace**

### AMQP 0.9.1 vs 1.0
- 0.9.1 — RabbitMQ 의 표준
- 1.0 — 다른 / 호환 X (대부분)

---

## 4. Kafka — Distributed Log

### 정의
- 2010 LinkedIn → Apache (2011)
- 분산 commit log
- 빅데이터 / streaming

### 핵심 개념

| 개념 | 설명 |
| --- | --- |
| **Topic** | log 의 이름 |
| **Partition** | topic 의 분할 (병렬) |
| **Offset** | partition 의 순서 ID |
| **Producer** | 메시지 publish |
| **Consumer** | 메시지 consume |
| **Consumer Group** | 같은 group — partition 분산 |
| **Broker** | Kafka 서버 |

### 동작
```
Topic: orders
  Partition 0: [msg1, msg2, msg3, ...]
  Partition 1: [msg1, msg2, ...]
  Partition 2: [msg1, ...]

Producer → key = "user-1" → hash → Partition 1
Consumer Group A: 
  Consumer 1 → Partition 0
  Consumer 2 → Partition 1
  Consumer 3 → Partition 2
Consumer Group B:
  Consumer X → 모든 partition (다른 group)
```

### Replication
- Partition 별 replica (보통 3)
- Leader + Followers
- Leader 죽으면 — follower 가 leader

### Retention
- 시간 기반 (1 일 / 7 일)
- 크기 기반 (1 TB)
- Log compaction (key 별 최신만)

### 보장
- **Per-partition** 순서
- **At-least-once** (기본)
- **Exactly-once** (transactional)

### 사용
- Event sourcing
- 로그 집계 (ELK, Logstash)
- Stream 처리 (Kafka Streams, Flink)
- CDC (Change Data Capture, Debezium)

### Schema Registry
- Avro / Protobuf / JSON Schema
- 메시지 schema 관리 / 검증

### KSQL / ksqlDB
- SQL on streams

---

## 5. NATS — 모던 / 빠른

### 정의
- Derek Collison (CloudFoundry 출신, 2010+)
- Go 작성 — 매우 빠름 (수백만 msg/s)
- Pub-Sub + Request-Response

### JetStream
- NATS 의 영속 stream
- Kafka 대체 lite

### 사용
- 마이크로서비스 (Kubernetes Service Mesh)
- IoT
- 게임

---

## 6. Pulsar — Apache (Yahoo)

### 정의
- 2016 Yahoo → Apache
- Kafka + RabbitMQ 의 통합 시도

### 특징
- BookKeeper (저장) + Pulsar (broker) 분리
- 멀티 테넌시
- 지오 복제 native

---

## 7. Cloud Native Messaging

### AWS
- **SQS** — Simple Queue (큐)
- **SNS** — Simple Notification (pub-sub)
- **Kinesis** — Kafka 대체
- **EventBridge** — event bus
- **MSK** — 매니지드 Kafka

### GCP
- **Pub/Sub** — fully managed
- **Pub/Sub Lite** — Kafka 대체

### Azure
- **Service Bus** (큐 / topic)
- **Event Hubs** — Kafka 대체
- **Event Grid**

---

## 8. 큐 vs 스트림

### Queue (RabbitMQ / SQS / AMQP)
- FIFO
- 한 메시지 → 한 consumer (workers)
- 처리 후 삭제

### Stream / Log (Kafka / Kinesis)
- Append-only
- 메시지 보관 (retention 까지)
- 여러 consumer 가 다른 offset
- Replay 가능

### 사용 분기
- 작업 분산 (job queue) — Queue
- Event sourcing / 분석 — Stream
- 둘 다 가능한 use case — Kafka (대규모)

---

## 9. Delivery Guarantees

| | 의미 |
| --- | --- |
| **At-most-once** | 0 또는 1 번 (loss 가능) |
| **At-least-once** | 1 번 이상 (중복 가능) |
| **Exactly-once** | 정확히 1 번 (어려움) |

### Exactly-once 의 실제
- 큐 / 스트림 — at-least-once 이 일반적
- Consumer 가 idempotent 하게 (같은 key → 같은 결과)
- Kafka — transactional producer + consumer offset commit

---

## 10. Dead Letter Queue (DLQ)

### 정의
- 처리 실패 메시지 별도 큐
- 재시도 한도 초과 시

### 흐름
```
Queue → Worker (fail) → Retry → Retry → DLQ → 사람 검토
```

### 사용
- 잘못된 메시지 빠른 격리
- 운영 / 디버깅

---

## 11. Backpressure

### 문제
- Producer 가 빠르면 — Consumer 못 따라옴 → 큐 폭증

### 해결
- Bounded queue (메모리 한계)
- Backpressure 신호 (Kafka — pause)
- Auto-scale consumer
- Rate limit producer

---

## 12. 함정

### 함정 1 — Kafka 의 partition 변경
Partition 늘리면 — 같은 key 다른 partition. Re-shuffle 필요.

### 함정 2 — Exactly-once 의 비용
Transactional — throughput ↓. 대부분 — at-least-once + idempotent.

### 함정 3 — RabbitMQ 의 unbounded queue
Disk full → broker 죽음. TTL / max-length 정책.

### 함정 4 — MQTT QoS 2 의 성능
2-phase — 4 packet per msg. QoS 1 권장.

### 함정 5 — Kafka consumer lag
처리 못 따라옴 — lag 폭증. 모니터 / scale.

### 함정 6 — Schema evolution
Avro / Protobuf — backward / forward compat. Schema Registry.

### 함정 7 — 모든 일 — Kafka
작은 시스템 — overkill. SQS / RabbitMQ 가 단순.

---

## 13. 학습 자료

- "Kafka: The Definitive Guide"
- "RabbitMQ in Action"
- "Designing Data-Intensive Applications" (Kleppmann)
- MQTT.org / AMQP 0-9-1 spec

---

## 14. 관련

- [[rpc-messaging]] — Hub
- [[grpc]] — sync vs async
- [[../topics/event-driven]] (TBD)
