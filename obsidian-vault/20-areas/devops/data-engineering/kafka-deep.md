---
title: "Kafka deep — 운영 / tuning / 확장"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T11:04:00+09:00
tags: [devops, data-engineering, kafka]
---

# Kafka deep — 운영 / tuning / 확장

**[[data-engineering|↑ data-engineering]]**

---

## 1. 무엇

```
log-based distributed message system:
  - high throughput (수십만 msg/s per broker)
  - durable (disk + replication)
  - ordered (per partition)
  - replay (offset 기반)
  - decoupling (producer ↔ consumer)
```

→ 가장 흔한 ingestion / streaming backbone.

---

## 2. broker / cluster

```
Cluster = N broker
N=3 권장 (RF=3, 1 broker fail OK)
N=5-9 = 큰 production

각 broker:
  - 일부 partition leader
  - 다른 partition follower (ISR)
  - heart-beat 으로 cluster 인지

control plane:
  ZooKeeper (legacy, Kafka < 3.3)
  KRaft (Kafka 3.3+, ★ 권장)
```

---

## 3. KRaft (★ 2024+ 표준)

```
Kafka 3.3+ 의 control plane:
  - ZooKeeper 의존성 제거
  - Raft consensus (broker 들 사이)
  - 더 빠른 metadata 작업
  - 운영 단순화

migration:
  - ZK 에서 KRaft 로 자동 (Kafka 3.5+)
  - 새 cluster = KRaft 만
```

---

## 4. broker tuning (★)

```properties
# server.properties
num.network.threads=8
num.io.threads=16
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600

# disk
log.dirs=/data/kafka1,/data/kafka2,/data/kafka3   # 여러 disk
log.flush.interval.messages=10000
log.flush.interval.ms=1000

# replication
default.replication.factor=3
min.insync.replicas=2
offsets.topic.replication.factor=3

# log retention
log.retention.hours=168                  # 7일
log.retention.bytes=-1                    # 무제한 (시간 기반)
log.segment.bytes=1073741824              # 1GB segment

# compression
compression.type=lz4                      # producer override 가능

# JVM
-Xmx6g -Xms6g                             # heap (rest = OS page cache)
-XX:+UseG1GC
-XX:+UnlockExperimentalVMOptions
-XX:+ParallelRefProcEnabled
-XX:+AlwaysPreTouch
```

---

## 5. partition design (★)

```
partition 수:
  너무 적음: throughput limit
  너무 많음: broker overhead
  
guideline:
  partition 수 ≥ consumer 수 (max parallel)
  partition 수 ≥ peak msg/s / 10MB (per partition throughput)
  
일반:
  작은 topic: 6 partition
  중간: 12-24
  큰: 24-100
  매우 큰: 100+
```

→ partition 추가 OK. 감소 불가.

→ key 의 hash 가 partition 결정. hot key 검토.

---

## 6. replication / ISR

```
RF (replication factor) = 3:
  leader + 2 follower

ISR (In-Sync Replicas):
  - 현재 leader 따라잡고 있는 replica 집합
  - min.insync.replicas=2 → ack=all 시 2 replica 확인 필요
  
broker fail 시:
  - ISR 중 새 leader 선출
  - 작동 계속
```

→ "RF=3 + min.insync.replicas=2" 가 표준.

---

## 7. producer 최적화

```java
// performance
props.put("compression.type", "lz4");          // lz4 / zstd 권장
props.put("batch.size", 65536);                 // 64KB
props.put("linger.ms", 20);                     // 20ms wait for batch
props.put("buffer.memory", 67108864);           // 64MB

// reliability
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);
props.put("max.in.flight.requests.per.connection", 5);
props.put("enable.idempotence", true);          // ★ exactly-once producer
```

```
trade-off:
  acks=0       → 빠름, lost 가능
  acks=1       → leader 만 → leader fail = lost
  acks=all     → 모든 ISR 확인 (★ 권장)
```

---

## 8. consumer 최적화

```java
// performance
props.put("fetch.min.bytes", 1024);            // min batch
props.put("fetch.max.wait.ms", 500);            // max wait for batch
props.put("max.poll.records", 500);
props.put("max.partition.fetch.bytes", 10485760);   // 10MB

// reliability
props.put("enable.auto.commit", "false");       // manual commit
props.put("auto.offset.reset", "earliest");      // 또는 latest
props.put("isolation.level", "read_committed");  // EOS

// rebalance
props.put("max.poll.interval.ms", 300000);       // 5min
props.put("session.timeout.ms", 30000);
props.put("heartbeat.interval.ms", 3000);
```

---

## 9. consumer rebalancing (★ 흔한 문제)

```
새 consumer join / leave / partition 추가:
  → group 안 모든 consumer 가 일시 정지 (stop the world)
  → 처리 lag 증가

mitigation:
  - cooperative rebalancing (incremental)
  - static membership (group.instance.id) — restart 시 같은 instance
  - max.poll.interval.ms 적절히 (너무 짧으면 rebalance 폭주)
```

```java
// cooperative
props.put("partition.assignment.strategy",
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
```

---

## 10. exactly-once semantics (EOS)

```
producer:
  enable.idempotence=true           # 중복 제거
  transactional.id="my-tx"          # transaction

consumer:
  isolation.level=read_committed    # commit 안 된 msg 안 봄

Kafka 안에서만 EOS.
외부 system (DB) 까지 = outbox / idempotency 별도.
```

---

## 11. tiered storage (★ 큰 비용 절감)

```
Kafka 3.6+ (KIP-405):
  hot data: 로컬 disk (빠른 access)
  cold data: S3 / GCS (저렴)

설정:
  remote.log.storage.system.enable=true
  remote.log.metadata.manager.class.name=...
  remote.storage.manager.class.name=...
  
local retention: 1일
total retention: 30일 (S3)
```

→ TB+ 의 long retention 비용 1/10.

---

## 12. monitoring (★)

```
metric (JMX → Prometheus):
  - broker:
    - request rate / time
    - under-replicated partitions
    - ISR shrink rate
    - active controller count
    - bytes in/out per topic
    - log size per partition
    - disk usage per broker
    
  - producer:
    - record send rate
    - record error rate
    - batch size avg
    
  - consumer:
    - records consumed rate
    - lag (★ 핵심)
    - rebalance rate
    - poll latency

도구:
  - kafka-exporter (Prometheus)
  - Kafka Manager / CMAK
  - Confluent Control Center
  - Conduktor
  - Kafka UI (OSS)
```

---

## 13. 운영 명령

```bash
# topic
kafka-topics --create --topic orders --partitions 12 --replication-factor 3
kafka-topics --list
kafka-topics --describe --topic orders
kafka-topics --alter --topic orders --partitions 24      # add only

# consumer group
kafka-consumer-groups --list
kafka-consumer-groups --describe --group order-processor
kafka-consumer-groups --reset-offsets --group X --topic Y --to-earliest --execute

# performance test
kafka-producer-perf-test --topic test --num-records 1000000 --record-size 1024 \
    --throughput -1 --producer-props bootstrap.servers=kafka:9092

kafka-consumer-perf-test --topic test --bootstrap-server kafka:9092 --messages 1000000

# JMX dump
kafka-broker-api-versions --bootstrap-server kafka:9092
```

---

## 14. 운영 best practice

```
☐ RF=3 + min.insync.replicas=2
☐ KRaft (or ZK 3.5+ migration)
☐ broker 3-9 (홀수)
☐ multi-AZ (rack awareness)
☐ disk = SSD (큰 IO)
☐ JVM heap 6-8GB (나머지 OS page cache)
☐ monitoring + alert (lag / disk / ISR)
☐ retention 정책
☐ tiered storage (TB+)
☐ schema registry (Avro)
☐ ACL / mTLS (보안)
☐ MirrorMaker 2 (DR)
☐ 정기 partition rebalance
☐ broker rolling upgrade
```

---

## 15. alternative

```
대안 (작은 / 다른 use case):
  - Redpanda — Kafka-compatible, Go, 빠름
  - Pulsar — multi-tenancy, geo-replication
  - NATS JetStream — lightweight
  - AWS Kinesis — managed
  - GCP Pub/Sub — managed
  - AWS MSK — managed Kafka (Kafka 그대로)
  - Confluent Cloud — Kafka SaaS

→ 일반 = Kafka. managed = MSK / Confluent. 빠름 = Redpanda.
```

---

## 16. 함정

1. **partition 1 / under-partitioned** — consumer scale 한계.
2. **partition 너무 많음** — broker overhead.
3. **RF=1** — broker fail = data loss.
4. **acks=1 + 외부 system** — leader fail 시 lost.
5. **manual commit 안 함 + auto** — at-most-once.
6. **rebalance 폭주** — max.poll.interval 너무 짧음.
7. **schema 호환성 없음** — consumer break.
8. **monitoring 없음** — lag 모름.

---

## 17. 관련

- [[data-engineering|↑ data-engineering]]
- [[../distributed-systems/messaging-patterns|↗ messaging]]
- [[real-time-streaming]]
- [[cdc-debezium]]
