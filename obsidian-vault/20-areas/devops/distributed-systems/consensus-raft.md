---
title: "Consensus — Raft / Paxos / Zab"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T10:39:00+09:00
tags: [devops, distributed-systems, consensus, raft]
---

# Consensus — Raft / Paxos / Zab

**[[distributed-systems|↑ distributed-systems]]**

---

## 1. 무엇 / 왜

```
"여러 node 가 한 값에 합의" 알고리즘.

용도:
  - leader election (어느 node 가 master?)
  - replicated log (모두 같은 순서)
  - distributed config (etcd / Consul / ZK)
  - distributed lock
  - cluster membership
```

→ 분산 DB / k8s / etcd / Consul 의 backbone.

---

## 2. 어디 쓰이나

| 시스템 | 알고리즘 |
| --- | --- |
| **etcd** (k8s) | Raft |
| **Consul** | Raft |
| **CockroachDB** | Raft |
| **TiDB** | Raft |
| **MongoDB** | Raft-like (replica set) |
| **k3s embedded** | Raft (etcd) |
| **Apache ZooKeeper** | Zab (Paxos-like) |
| **Apache Kafka** (KRaft) | Raft |
| **Cassandra** | gossip + LWW (consensus 없음) |
| **Google Spanner** | Paxos + TrueTime |

→ **Raft 가 modern 표준** (Paxos 보다 이해 쉬움).

---

## 3. Raft 의 핵심

```
3 역할:
  Follower    수동, leader 의 명령 받음
  Candidate   election 도전
  Leader      모든 write 처리

phase:
  1. Leader Election
  2. Log Replication
  3. Safety
```

---

## 4. Leader Election

```
초기 / leader fail 후:
  
1. follower 들 timeout (150-300ms random)
2. timeout 만료 → candidate 됨
3. term ++ , vote 자신 → 다른 node 들에 RequestVote
4. majority vote 받음 → leader
5. heartbeat 보냄
6. 다른 node 들은 leader 인지

가장 자주 첫 timeout 만료한 node 가 leader.
random timeout = split vote 가능성 ↓.
```

```
node A (term 5): "투표해 줘"
node B: "OK"
node C: "OK"
node A: majority → 나는 leader. term 5.
```

---

## 5. Log Replication

```
client → leader: "x = 5"

leader:
  1. log entry (term=5, idx=10, "x=5") 자기 log 에 append
  2. AppendEntries RPC 모든 follower
  3. majority ack 받음 → commit
  4. apply to state machine
  5. follower 들도 commit (다음 heartbeat 시)
  6. client 에 response

→ majority 가 받으면 안전 (single node fail 견딤).
```

---

## 6. safety property

```
1. Election safety:    한 term 에 leader 한 명만
2. Leader append-only: leader 가 자기 log 절대 안 지움
3. Log matching:       두 log 의 같은 idx + term ⇒ 그 전 모두 같음
4. Leader completeness: 한 entry 가 commit 되면 모든 future leader 에 있음
5. State machine safety: 같은 idx apply ⇒ 모두 같은 결과
```

---

## 7. quorum (★)

```
N = 3:  majority = 2  (1 fail OK)
N = 5:  majority = 3  (2 fail OK)
N = 7:  majority = 4  (3 fail OK)

짝수 (4) = 홀수 (3) 와 동일 fault tolerance 인데 비쌈.

→ 항상 홀수.
```

```
network partition:
  3 node split → (2,1)
  majority(2) 쪽 → 동작
  minority(1) 쪽 → leader 없음, write reject
  
split-brain 방지!
```

---

## 8. split-brain (★)

```
잘못된 시스템:
  partition → 양쪽 다 active → 둘 다 write
  → merge 시 conflict

Raft 의 보호:
  partition 발생 → majority 한쪽 만 leader
  minority 의 leader 가 새 client 받아도 commit 못 함 (majority X)
  → safe
```

---

## 9. etcd (★ k8s 의 store)

```bash
# 설치
etcd --name node1 \
     --listen-client-urls http://0.0.0.0:2379 \
     --advertise-client-urls http://10.0.0.1:2379 \
     --listen-peer-urls http://0.0.0.0:2380 \
     --initial-advertise-peer-urls http://10.0.0.1:2380 \
     --initial-cluster node1=http://10.0.0.1:2380,node2=http://10.0.0.2:2380,node3=http://10.0.0.3:2380 \
     --initial-cluster-token my-cluster \
     --initial-cluster-state new

# health
etcdctl endpoint health --cluster

# leader 확인
etcdctl endpoint status --write-out=table

# member
etcdctl member list
```

```bash
# 사용
etcdctl put /key1 value1
etcdctl get /key1
etcdctl watch /key1

# lease (TTL)
etcdctl lease grant 60
etcdctl put /key2 value2 --lease=<id>

# transaction (compare-and-swap)
etcdctl txn <<EOF
mod("/lock") = "0"

put /lock "1"

get /lock
EOF
```

---

## 10. ZooKeeper (★ legacy)

```
Apache ZooKeeper:
  - Zab protocol (Paxos-like)
  - 거의 etcd 와 같은 역할
  - Hadoop / Kafka (legacy) / Solr / HBase
  - znode (file-like) hierarchy
  - watch
  - ephemeral / sequential znode

Kafka 3.3+: KRaft → ZK 의존성 제거.
```

---

## 11. distributed lock (★)

```python
# etcd v3 lock
import etcd3

client = etcd3.client(host='etcd', port=2379)
with client.lock('my-lock', ttl=60) as lock:
    # critical section
    do_work()
# 자동 release
```

```bash
# etcdctl
etcdctl lock /my-lock -- /bin/sleep 10
```

→ etcd 의 lease + transaction 으로 atomic lock.

---

## 12. leader election 사용

```go
// Go etcd client
import "go.etcd.io/etcd/client/v3/concurrency"

session, _ := concurrency.NewSession(client, concurrency.WithTTL(15))
defer session.Close()

election := concurrency.NewElection(session, "/my-election")

// 후보 (block 또는 비동기)
err := election.Campaign(ctx, "node-1")
// 여기까지 오면 = leader

// leader 작업
runLeaderTask()

// 양보
election.Resign(ctx)
```

→ 분산 cron / scheduler / single-writer 패턴.

---

## 13. 성능 / 한계

```
etcd:
  - 1000 write/s OK
  - 큰 cluster (10+ node) 부담 (heartbeat 폭주)
  - per-key < 1.5MB
  - 총 storage 8GB 권장

→ 작은 metadata 적합. 큰 data store X.
```

---

## 14. Byzantine fault

```
Raft / Paxos = crash failure 가정
  (node 가 죽음, lie 안 함)

Byzantine fault = node 가 lie / 잘못된 data
  → PBFT / Tendermint / Tangaroa

일반 회사 = crash 만 가정 OK.
blockchain = Byzantine 필수.
```

---

## 15. 함정

1. **2 node cluster** — 1 fail = quorum 잃음.
2. **slow disk** — Raft 의 fsync latency 가 throughput 결정.
3. **network slow** — heartbeat timeout → 잦은 election.
4. **etcd 큰 value** — < 1.5MB. config 만.
5. **leader election 의 split vote** — random timeout 의 변동.
6. **member 추가/제거 잘못** — quorum 일시 잃음.
7. **backup 안 함** — etcd snapshot 정기.

---

## 16. 관련

- [[distributed-systems|↑ distributed-systems]]
- [[leader-election]]
- [[distributed-lock]]
- [[../k3s/ha-mode|↗ k3s HA]]
