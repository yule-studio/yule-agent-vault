---
title: "L4 Load Balancing — TCP / UDP"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T06:05:00+09:00
tags:
  - network
  - load-balancing
  - l4
  - tcp
---

# L4 Load Balancing — TCP / UDP

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | NLB / IPVS / DSR / Maglev |

**[[load-balancing|↑ Load Balancing]]** · **[[../network|↑↑ network hub]]**

---

## 1. 한 줄 정의

**TCP / UDP 헤더만 보고** 부하 분산. L7 보다 빠르지만 HTTP 헤더 / URL 기반 라우팅 불가.

---

## 2. 동작

```
Client → LB: TCP SYN to vip:443
LB: backend 선택 (5-tuple hash)
LB → Backend: SYN forward (NAT 또는 DSR)
Backend → Client: SYN-ACK
... (한 connection — 같은 backend)
```

### 5-tuple
- src IP / src port / dst IP / dst port / protocol

→ 같은 5-tuple — 같은 backend (sticky by nature).

---

## 3. NAT vs DSR

### NAT (Network Address Translation)
```
Client → LB (vip): src 1.2.3.4, dst 10.0.0.1
LB → Backend:     src 1.2.3.4, dst 10.0.0.2  (DNAT)
Backend → LB:     src 10.0.0.2, dst 1.2.3.4
LB → Client:      src 10.0.0.1, dst 1.2.3.4  (SNAT 또는 그대로)
```

- 모든 패킷 LB 통과
- 양방향 부하

### DSR (Direct Server Return)
```
Client → LB (vip): TCP packet
LB → Backend: forward (MAC rewrite, IP 그대로)
Backend → Client: 직접 응답 (loopback 에 vip 설정)
```

- 응답이 LB 우회 → 부하 ↓ (1/10 정도)
- Backend 가 vip 가진 척 (loopback alias)

### 결정
- HTTP / small RPC — NAT (대부분)
- 스트리밍 / 대용량 — DSR

---

## 4. ECMP — Equal-Cost Multi-Path

### 의미
- 라우터가 같은 cost 의 여러 경로 — 5-tuple hash 로 분산

### LB 자체의 분산
```
Internet
   ↓
Router (ECMP)
 ↙ ↓ ↘
LB1 LB2 LB3        ← 모두 같은 anycast IP
 ↓   ↓   ↓
Backend pool
```

→ Maglev (Google), Katran (Facebook), GLB (GitHub) — XDP / DPDK 기반.

---

## 5. IPVS — Linux Kernel L4

### 정의
- Linux Virtual Server — kernel 의 L4 LB
- LVS 의 일부 (kernel 2.4+)

### 모드
- **NAT** — 게이트웨이
- **DR (Direct Routing / DSR)** — 같은 L2 망
- **TUN (IP tunneling)** — 다른 망

### 설정
```bash
ipvsadm -A -t 10.0.0.1:80 -s rr      # Virtual service
ipvsadm -a -t 10.0.0.1:80 -r 10.0.0.2:80 -g
ipvsadm -a -t 10.0.0.1:80 -r 10.0.0.3:80 -g
```

### 사용
- Kubernetes kube-proxy (ipvs mode)
- Keepalived 와 결합

---

## 6. AWS NLB — Network Load Balancer

### 특징
- L4 (TCP / UDP / TLS pass-through)
- 매우 빠름 (millions QPS)
- 정적 IP (Elastic IP)
- DSR 아님 — NAT 형식

### 사용
- 게임 / VoIP / IoT (UDP)
- gRPC / WebSocket
- 매우 낮은 latency

---

## 7. Maglev — Google 의 L4 (2008-)

### 정의
- Google 의 대규모 L4 LB
- ECMP 기반 — 여러 Maglev 가 같은 vip
- Consistent hashing — 백엔드 변경 시 영향 최소

### 효과
- TB/s 단위 트래픽
- 1 ms latency
- Backend 추가 / 제거 — connection 유지

### 논문
- "Maglev: A Fast and Reliable Software Network Load Balancer" (2016)

---

## 8. Katran — Facebook

### 정의
- XDP / eBPF 기반 L4 LB
- Linux kernel — XDP_TX (드라이버 가까이)

### 효과
- 1 server — 10M+ pps
- Userspace 우회

---

## 9. GLB — GitHub

### 정의
- 두 단계 — 첫 LB (consistent hash) → "Director"
- Director 가 (real) backend forward

### 효과
- LB 가 죽어도 — 다른 LB 가 같은 결정
- "tier 1 / tier 2" 구조

---

## 10. Proxy Protocol (L4 의 클라이언트 IP)

### 문제
- L4 NAT 가 LB IP 로 변환 → backend 가 클라이언트 IP 모름

### Proxy Protocol
```
PROXY TCP4 1.2.3.4 5.6.7.8 12345 443\r\n
<원래 TCP 데이터>
```

- HAProxy 가 만든 표준
- AWS NLB / Envoy / nginx 지원

자세히 → [[load-balancing]]

---

## 11. UDP Load Balancing

### 도전
- Connectionless — 5-tuple 기반 sticky 가 약함
- 패킷 단위 분산 — 응답 순서 보장 X

### QUIC (HTTP/3)
- Connection ID 기반 라우팅 (5-tuple 변경 시도 같은 conn)
- LB 가 Connection ID parsing 필요

자세히 → [[../quic/quic]]

---

## 12. 함정

### 함정 1 — Long-lived connection
WebSocket / gRPC — 한 connection 이 한 backend.
Backend scale-down 시 — connection 끊김 가능.

### 함정 2 — Health check 의 false-positive
TCP-level check 만 — backend 의 application down 인지 모름.
HTTP-level health 권장 (L7).

### 함정 3 — DSR 의 backend 설정
backend 의 loopback 에 vip alias 필요.
ARP 응답 막아야 (arp_ignore / arp_announce).

### 함정 4 — Cross-zone disabled (AWS)
NLB — cross-zone 끄면 — zone 별 분리. zone 별 backend 적으면 불균형.

### 함정 5 — Source IP preservation
NAT 기본 — backend 가 LB IP 봄. Proxy Protocol / 사용자 헤더 필요.

---

## 13. 학습 자료

- "Maglev" Google paper
- "Katran" GitHub repo (Facebook)
- "GLB" GitHub blog
- IPVS docs (Linux)
- AWS NLB docs

---

## 14. 관련

- [[load-balancing]] — Hub
- [[l7-load-balancing]] — 비교
- [[load-balancing-algorithms]] — 알고리즘
- [[../tcp/tcp]] — 5-tuple
