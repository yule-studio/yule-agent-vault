---
title: "OS Topics (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:20:00+09:00
tags:
  - operating-system
  - topic
  - hub
---

# OS Topics (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 면접 / 시스템 토픽 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 주제별 깊이

| 주제 | 노트 |
| --- | --- |
| C10K / C10M | [[c10k-c10m]] |
| OOM 시나리오 | [[oom-scenarios]] |
| Thrashing | [[thrashing]] |
| Cache Locality | [[cache-locality]] |
| Context Switch 비용 | [[../scheduling/context-switch]] |
| Linux vs Windows vs macOS | [[linux-vs-others]] |

---

## 2. 면접 빈출

### 2.1 Process vs Thread
자세히 → [[../process/process#8-프로세스-vs-스레드]], [[../threads/threads#2-프로세스-vs-스레드]]

### 2.2 Deadlock 4 조건
자세히 → [[../synchronization/deadlock]]

### 2.3 Virtual Memory
자세히 → [[../memory/virtual-memory]]

### 2.4 fork vs exec
자세히 → [[../process/fork-exec]]

### 2.5 select / poll / epoll
자세히 → [[../io/select-poll-epoll]]

### 2.6 fsync 의 의미
자세히 → [[../filesystem/fsync-durability]]

### 2.7 VM vs Container
자세히 → [[../virtualization/vm-vs-container]]

### 2.8 OOM Killer
자세히 → [[../memory/swap-oom]]

---

## 3. 시스템 설계 토픽

### 3.1 1억 동시 접속 (C10M)
자세히 → [[c10k-c10m]]

### 3.2 zero-downtime deploy
- graceful shutdown (SIGTERM → drain → SIGKILL after timeout)
- health check + load balancer
- rolling update
- blue-green
- canary

### 3.3 zero-copy 의 전제
- sendfile / splice / mmap
- NIC 의 SG-DMA 지원
- kTLS
- 작은 파일 = 효과 ↓

자세히 → [[../io/zero-copy]]

### 3.4 LB 의 worker 모델
- prefork (Apache 옛)
- thread per request (Tomcat 옛)
- event loop (Nginx, Node)
- N:M / coroutine (Go, Java Loom)

### 3.5 cgroup CPU limit + tail latency
- throttle → P99 spike
- limit 조정 또는 제거

자세히 → [[../virtualization/cgroups]]

---

## 4. 관련

- [[../operating-system|↑ OS hub]]
