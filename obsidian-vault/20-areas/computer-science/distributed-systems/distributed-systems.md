---
title: "분산 시스템 (Distributed Systems)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T11:30:00+09:00
tags:
  - distributed-systems
  - consensus
  - cap
  - replication
  - consistency
---

# 분산 시스템 (Distributed Systems)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**여러 컴퓨터가 네트워크로 협력** 해 하나의 시스템처럼 동작. 부분 실패 (partial
failure), 비동기 통신, 시계 불일치가 본질적 어려움.

---

## 2. 분산 시스템의 8 가지 오류

Peter Deutsch (Sun, 1994) 가 명명한 **"Fallacies of Distributed Computing"**:

1. 네트워크는 신뢰할 수 있다 — ❌ (패킷 손실 / 지연)
2. 지연 시간이 0 이다 — ❌
3. 대역폭은 무한하다 — ❌
4. 네트워크는 안전하다 — ❌
5. 토폴로지는 바뀌지 않는다 — ❌ (노드 추가 / 장애)
6. 관리자가 한 명이다 — ❌
7. 전송 비용이 0 이다 — ❌
8. 네트워크는 동질하다 (homogeneous) — ❌

모든 분산 시스템 설계자는 이 오류들에 매번 부딪힌다.

---

## 3. 시스템 모델

### 3.1 동기성 모델

- **Synchronous** — 메시지 지연 & 처리 시간 상한 보장
- **Partially Synchronous** — 평소엔 synchronous, 가끔 비동기
- **Asynchronous** — 보장 없음 (가장 일반)

### 3.2 장애 모델

- **Crash failure** — 노드가 멈춤
- **Omission failure** — 메시지 누락
- **Byzantine failure** — 임의 행동 (악의 / 버그)

### 3.3 FLP 정리 (Fischer-Lynch-Paterson, 1985)

**비동기 + crash failure 환경에서는 결정적 합의 불가능**.

해결법: 부분 동기 가정 (timeout), 무작위성, failure detector.

---

## 4. CAP & PACELC

[[../database-theory/database-theory#9 CAP 정리 & PACELC]] 참조.

### 4.1 일관성 (Consistency) 수준

| 수준 | 보장 | 예 |
| --- | --- | --- |
| **Linearizable** | 실시간 순서 보장 | Spanner, etcd |
| **Sequential** | 프로세스 별 순서만 | 일부 시스템 |
| **Causal** | 인과 관계 보존 | COPS, Vector clock |
| **Eventual** | 결국 수렴 | DynamoDB, Cassandra |

### 4.2 Linearizability

- "마치 단일 노드처럼 보임"
- 모든 연산이 시점에 매핑됨
- 가장 강한 일관성. 비싸다 — 동기 복제 + 합의.

---

## 5. 시계 (Clock)

### 5.1 물리 시계

- NTP 동기화 — ±50ms 이내
- 그래도 노드 간 drift, leap second
- "실제 시간" 으로 정렬 불가능

### 5.2 Lamport Clock

- 각 노드의 카운터
- 메시지 보낼 때 `max(local, received) + 1`
- "happens-before" 관계 보존
- 동시성 (concurrent) 구분 불가

### 5.3 Vector Clock

- 각 노드별 카운터 벡터
- "happens-before" + "concurrent" 모두 식별
- 크기 = 노드 수 (확장성 문제)

### 5.4 Hybrid Logical Clock (HLC)

- 물리 시간 + 논리 카운터
- CockroachDB, MongoDB 사용

### 5.5 Google TrueTime

- GPS + 원자 시계로 ±7ms 이내 보장
- 명시적 불확실성 구간 `TT.now()` 반환
- Spanner 의 globally consistent 가능케 함

---

## 6. 합의 알고리즘 (Consensus)

### 6.1 합의가 풀어야 하는 것

여러 노드가 **하나의 값** 에 동의:
- Agreement — 모두 같은 값
- Validity — 결정 값은 누군가 제안한 값
- Termination — 결국 결정함

### 6.2 Paxos (Lamport 1989)

- 첫 분산 합의 알고리즘
- 어려움으로 악명 — "Paxos Made Simple" 논문도 어려움
- Phase 1 (Prepare) + Phase 2 (Accept)

### 6.3 Raft (Ongaro & Ousterhout 2014)

- "이해 가능한 합의" 가 설계 목표
- 강한 leader + log replication + leader election
- etcd, Consul, TiKV 가 사용

```
1. Leader election — 노드 중 하나가 leader (term)
2. Log replication — leader 가 entry 받아 follower 에 복제
3. Commit — 과반 복제 시 commit
4. Apply — state machine 에 적용
```

### 6.4 Multi-Paxos / Zab (ZooKeeper)

- Paxos 변형
- ZooKeeper, Chubby (Google)

### 6.5 PBFT / HotStuff

- Byzantine fault tolerance
- 블록체인의 합의 기반

### 6.6 Quorum

- N 노드 중 W 쓰기 ACK / R 읽기 ACK
- `W + R > N` → 일관성 보장
- 예) N=3, W=2, R=2 → 강한 일관성

---

## 7. 복제 (Replication)

### 7.1 패턴

- **Single-leader (Primary-Replica)** — 가장 흔함
- **Multi-leader** — 데이터센터 간, 충돌 해결 필요
- **Leaderless** — Dynamo / Cassandra, quorum 기반

### 7.2 동기 vs 비동기

- **Sync** — 모든 replica 응답 후 commit. 안전, 느림.
- **Async** — leader 만 commit. 빠름, 손실 위험.
- **Semi-sync** — N 중 일부 sync.

### 7.3 충돌 해결

- **Last-Write-Wins** — timestamp 기준 (시계 문제)
- **Multi-value** — 모든 버전 보관, 클라이언트가 해결
- **CRDT** — 자동 병합 가능한 자료구조

### 7.4 CRDT (Conflict-free Replicated Data Type)

- **G-Counter** — 증가만 가능
- **PN-Counter** — 증가 + 감소
- **OR-Set** — 추가 + 삭제
- **LWW-Register** — 시간 기반
- **RGA** (Replicated Growable Array) — 협업 편집 (Google Docs / Figma)

---

## 8. 분산 트랜잭션

### 8.1 2PC (Two-Phase Commit)

```
Coordinator → Prepare → Participants
                ↓ (모두 Yes?)
Coordinator → Commit / Abort → Participants
```

**문제**: coordinator 장애 시 blocking. CAP 의 P 부족.

### 8.2 3PC

- 추가 단계로 non-blocking 시도
- 실제로는 잘 안 씀

### 8.3 Saga 패턴

- 긴 트랜잭션을 작은 로컬 트랜잭션 + 보상 (compensating) 액션 체인
- **Choreography** — 이벤트 기반 (각 서비스가 결정)
- **Orchestration** — 중앙 코디네이터

### 8.4 TCC (Try-Confirm-Cancel)

- Try — 자원 예약
- Confirm — 최종 적용
- Cancel — 예약 해제

### 8.5 Outbox 패턴

- DB 트랜잭션에 메시지를 outbox 테이블에 함께 저장
- 별도 프로세스가 메시지 큐에 발행
- "exactly-once" 효과

---

## 9. 메시지 큐 / 스트리밍

### 9.1 메시지 큐

- **RabbitMQ** — AMQP, 라우팅 풍부
- **ActiveMQ** — JMS
- **AWS SQS** — managed
- **Redis Streams** — 가벼움

### 9.2 스트리밍 플랫폼

- **Kafka** — 분산 로그, 파티션, consumer group
- **Pulsar** — Kafka 대안, 멀티 테넌시
- **Kinesis** — AWS

### 9.3 전송 보장

| 수준 | 의미 |
| --- | --- |
| **At-most-once** | 손실 가능 |
| **At-least-once** | 중복 가능 — idempotent 필수 |
| **Exactly-once** | Kafka transactions, Outbox |

### 9.4 Pub-Sub

- Producer ⇒ Topic ⇒ Subscribers
- 1-to-N, 비동기, decoupling

---

## 10. 분산 패턴

### (a) Service Discovery
- Consul, etcd, ZooKeeper
- 클라이언트 사이드 (Ribbon) vs 서버 사이드 (Nginx)

### (b) Circuit Breaker (Netflix Hystrix)
- 실패율 임계치 넘으면 차단 → fallback
- Open → Half-Open → Closed

### (c) Bulkhead
- 자원 격리 — 한 부분 실패가 전체로 번지지 않음

### (d) Retry + Backoff
- 지수 백오프 + jitter
- "Thundering herd" 방지

### (e) Rate Limiting
- Token Bucket, Leaky Bucket
- Redis 로 분산 RL

### (f) Sidecar
- 부가 기능을 별도 컨테이너 (Envoy, Istio proxy)

### (g) CQRS (Command Query Responsibility Segregation)
- 쓰기 / 읽기 모델 분리

### (h) Event Sourcing
- 상태 대신 이벤트 로그 저장
- 시간 여행, 감사

---

## 11. 마이크로서비스

### 11.1 모놀리식 vs 마이크로서비스

| 측면 | 모놀리식 | 마이크로서비스 |
| --- | --- | --- |
| 배포 | 단일 | 독립 |
| 기술 스택 | 통일 | 다양 |
| 통신 | in-process | 네트워크 |
| 데이터 | 공유 DB | 분리 DB |
| 복잡도 | 낮음 | 높음 |
| 팀 자율성 | 낮음 | 높음 |

### 11.2 마이크로서비스 도전

- 분산 트랜잭션
- 데이터 일관성
- 디버깅 (분산 추적 — Jaeger, Zipkin)
- 운영 복잡도

### 11.3 Conway's Law

> "시스템 설계는 조직 구조를 반영한다"

마이크로서비스는 팀 단위 자율성과 깊이 연관.

---

## 12. 관찰 가능성 (Observability)

세 기둥:

### 12.1 Logs
- 구조화 (JSON), 중앙 집중 (ELK, Loki)
- Trace ID 로 상관 관계

### 12.2 Metrics
- Prometheus + Grafana
- RED / USE 방법론

### 12.3 Traces
- OpenTelemetry 표준
- 분산 trace ID 전파
- Jaeger, Zipkin, Datadog APM

---

## 13. 함정 / 안티패턴

### 함정 1 — 분산 모놀리스
서비스 분리했지만 강결합 → 모든 단점만 모음.

### 함정 2 — 동기 호출 체인
A → B → C → D, 한 곳만 느려져도 전체 지연.

### 함정 3 — 시간 의존
NTP 가 ms 단위 동기 보장 안 함. 시계 비교 X.

### 함정 4 — 정확히 한 번 (exactly-once) 환상
이상적이지만 실제론 idempotent + at-least-once 가 현실.

### 함정 5 — Split brain
네트워크 분할 시 양쪽이 모두 leader. Quorum 으로 방지.

### 함정 6 — 분산 캐시 일관성
TTL 만으로 부족. Cache invalidation 의 어려움.

### 함정 7 — 재시도 폭풍 (retry storm)
실패 시 모두 재시도 → 더 큰 부하. Backoff + jitter.

### 함정 8 — 메시지 순서 가정
큐는 종종 순서 보장 안 함. 파티션 키 / Sequence number.

---

## 14. 면접 / 시스템 설계 질문

1. **Netflix / YouTube / Twitter 설계**.
2. **URL Shortener** — Hash, Base62, Sharding.
3. **분산 캐시** — Consistent Hashing, Cache aside.
4. **Rate Limiter** — Token Bucket / Sliding Window.
5. **Distributed Lock** — Redlock, ZooKeeper.
6. **Chat App** — WebSocket, pub-sub, presence.
7. **Real-time leaderboard** — Sorted Set (Redis ZSET).
8. **Notification system** — Fan-out, push vs pull.
9. **Pastebin / Dropbox** — Object storage, chunking.
10. **Search Engine** — Inverted index, ranking.

---

## 15. 학습 자료

- **Designing Data-Intensive Applications** (Kleppmann) — 분산 시스템 바이블
- **Distributed Systems** (van Steen / Tanenbaum) — 무료 PDF
- **Patterns of Distributed Systems** (Unmesh Joshi, Martin Fowler) — https://martinfowler.com/articles/patterns-of-distributed-systems/
- **MIT 6.824** — 분산 시스템 강의 (YouTube)
- **The Morning Paper** (Adrian Colyer) — 분산 시스템 논문 리뷰
- Lamport 1978 — "Time, Clocks, and the Ordering of Events"
- Raft 논문 — "In Search of an Understandable Consensus Algorithm"

---

## 16. 관련

- [[../network/network]] — TCP/IP, HTTP, gRPC
- [[../database-theory/database-theory]] — CAP, MVCC
- [[../security-theory/security-theory]] — 인증, mTLS
- [[../operating-system/operating-system]] — IPC, Socket
- [[../computer-science|↑ computer-science]]
