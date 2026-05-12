---
title: "RIP — Routing Information Protocol"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T19:30:00+09:00
tags:
  - network
  - routing
  - rip
  - distance-vector
---

# RIP — Routing Information Protocol

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | RIPv1/v2/RIPng / Distance Vector / Bellman-Ford |

**[[routing|↑ Routing]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **분류** | Distance Vector / IGP |
| **알고리즘** | Bellman-Ford |
| **메트릭** | Hop count (최대 15) |
| **광고 주기** | 30 초 |
| **Convergence** | 느림 (분 단위) |
| **포트** | UDP 520 (RIPv1/v2), UDP 521 (RIPng IPv6) |
| **AD (Cisco)** | 120 |
| **RFC** | 1058 (v1), 2453 (v2), 2080 (RIPng) |

---

## 1. 한 줄 정의

가장 단순한 동적 라우팅 프로토콜. **Distance Vector** — "이웃에게서 들은 거리"
기반. 1988 RFC 1058 — 인터넷 초기 표준.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1969 | ARPANET — DV 알고리즘 사용 |
| 1982 | BSD `routed` 데몬 |
| 1988 | **RFC 1058 (RIPv1)** |
| 1994 | **RFC 1723 / 2453 (RIPv2)** — VLSM, 인증 |
| 1997 | **RFC 2080 (RIPng)** — IPv6 |

---

## 3. Distance Vector 원리

```
각 라우터:
1. 자기 인접 네트워크 알고 있음
2. 30 초마다 이웃에게 자기 라우팅 테이블 광고
3. 받은 정보 + 1 hop 으로 자기 테이블 갱신
4. 변화 있으면 다시 광고
```

**Bellman-Ford** 알고리즘 — 분산 (distributed) 구현.

```
d(v) = min { d(u) + cost(u, v) }  for all neighbor u
```

---

## 4. RIPv1 (RFC 1058)

### 4.1 한계
- **Classful** — Subnet mask 광고 X (8/16/24 만)
- **VLSM 미지원**
- **Broadcast** (255.255.255.255)
- 인증 X

→ 1990 년대 초 한계 명확 → v2.

---

## 5. RIPv2 (RFC 2453, 1994)

### 5.1 개선
- **VLSM** (Variable Length Subnet Mask) 지원
- **Classless** (CIDR)
- **Multicast** 224.0.0.9 (broadcast 대신)
- **MD5 인증**
- Next hop 명시 가능

### 5.2 메시지 형식

```
[Command (1)][Version (1)][Reserved (2)]
[ Address Family (2)][Route Tag (2)]
[IP Address (4)]
[Subnet Mask (4)]
[Next Hop (4)]
[Metric (4)]
... (최대 25 routes per message)
```

---

## 6. 메트릭 — Hop Count

```
직접 연결 = 0 (또는 1)
1 라우터 통과 = 1
2 라우터 통과 = 2
...
15 = 최대 (도달 가능)
16 = 무한대 (도달 불가)
```

### 한계 — 16 Hop
- 작은 망만 가능
- 큰 망에선 부적합 → OSPF/EIGRP

### Cost 와 무관
- 1 Gbps 광 vs 56 kbps 모뎀 — 같은 hop 으로 취급
- → 비효율적 경로 선택 가능

---

## 7. 동작 — Update

### 7.1 주기적 (30 초)
- 모든 인접 라우터에 전체 라우팅 테이블 광고

### 7.2 트리거 (Triggered) Update
- 변화 발생 시 즉시
- Convergence 가속

### 7.3 Invalid Timer (180 초)
- 라우트 받지 못한 시간 — 도달 불가 표시

### 7.4 Flush Timer (240 초)
- Invalid 후 60 초 더 → 테이블에서 제거

→ 변화 인식까지 **수 분** 소요. 모던 망에선 너무 느림.

---

## 8. Loop Prevention

### 8.1 Counting to Infinity 문제

```
A ─ B ─ C ─ Network
A 의 link 끊김
B 는 모름 → "A 통해 도달" 광고 (hop 2)
C 가 받음 → "B 통해 도달" 광고 (hop 3)
B 가 받음 → "C 통해 도달" 광고 (hop 4)
...
→ 무한 hop 증가
```

### 8.2 방어

#### Split Horizon
- 받은 인터페이스로 광고 X

#### Poison Reverse
- 받은 인터페이스로 metric 16 광고 ("절대 가지 마")

#### Triggered Update + Hold-Down Timer
- 변화 후 일정 시간 다른 광고 무시

#### Max Metric 16
- 16 = 무한 → counting 빠르게 종료

---

## 9. RIPng (IPv6, RFC 2080)

### 차이점
- IPv6 주소
- UDP 521
- Multicast `ff02::9`
- IPsec 인증 (RIPv2 의 MD5 대체)

---

## 10. Cisco 설정 예

```
router rip
  version 2
  network 192.168.1.0
  network 10.0.0.0
  no auto-summary           # CIDR 정확히
  passive-interface default # 모든 인터페이스 광고 X
  no passive-interface Gi0/0
```

### Linux quagga / FRRouting

```
router rip
  version 2
  network 192.168.1.0/24
  redistribute connected
  redistribute static
```

---

## 11. RIP vs OSPF 비교

| 측면 | RIP | OSPF |
| --- | --- | --- |
| 알고리즘 | DV (Bellman-Ford) | LS (Dijkstra) |
| 메트릭 | Hop count | Cost (BW) |
| Convergence | 분 단위 | 초 단위 |
| 망 크기 | 작음 (15 hop) | 큰 |
| CPU 사용 | 낮음 | 중간 |
| 대역폭 | 전체 테이블 광고 | 변화만 |
| 인증 | MD5 | MD5/SHA |
| 복잡도 | 단순 | 복잡 |

→ **RIP 는 사실상 사장**. 학습 목적 / 작은 lab 에서만.

---

## 12. 함정

### 함정 1 — Hop count 만 보고 선택
초고속 vs 저속 link 구분 X.

### 함정 2 — 30 초 광고
변화 인식 느림. Trigger 의존.

### 함정 3 — Counting to Infinity
Split Horizon 비활성 시 발생.

### 함정 4 — 16 hop 한계
거의 모든 현대 망 초과.

### 함정 5 — 인증 없는 RIPv1
임의 라우터가 가짜 광고 → MITM.

### 함정 6 — Auto-summary
CIDR 환경에서 의도치 않은 경로.

---

## 13. 학습 자료

- RFC 1058 (RIPv1), RFC 2453 (RIPv2), RFC 2080 (RIPng)
- **TCP/IP Illustrated Vol.1** Ch. 10
- **Routing TCP/IP Vol.1** (Doyle)
- Cisco "Routing Information Protocol"

---

## 14. 관련

- [[ospf]] — RIP 의 대체
- [[eigrp]] — DV 의 진화
- [[bgp]] — EGP / Path Vector
- [[routing]] — 라우팅 hub
