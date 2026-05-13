---
title: "Load Balancing — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T06:00:00+09:00
tags:
  - network
  - load-balancing
---

# Load Balancing — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | L4 / L7 / 알고리즘 / Health Check |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/osi-7-layer|↑↑ OSI]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **계층** | L4 (TCP/UDP) / L7 (HTTP) |
| **제품** | HAProxy / Nginx / Envoy / AWS ALB-NLB / GCP LB |
| **표준** | RFC 7239 (Forwarded), Proxy Protocol |
| **목적** | 부하 분산 / 가용성 / 확장성 |

---

## 1. 한 줄 정의

**여러 백엔드로 트래픽 분산** — 부하 분산 / 가용성 / 확장.

---

## 2. 계층 — L4 vs L7

| | L4 (Transport) | L7 (Application) |
| --- | --- | --- |
| **참조** | TCP / UDP 헤더 (IP, Port) | HTTP 헤더 (URL, Cookie, Host) |
| **속도** | 빠름 | 보통 (TLS 종료 시 부담) |
| **결정** | Connection 별 | Request 별 |
| **TLS** | passthrough | 종료 가능 |
| **재시도** | X | Possible |
| **제품** | NLB, IPVS | ALB, Nginx, Envoy |

자세히 → [[l4-load-balancing]], [[l7-load-balancing]]

---

## 3. 알고리즘 — 한눈

자세히 → [[load-balancing-algorithms]]

| 알고리즘 | 용도 |
| --- | --- |
| Round-Robin | 가장 단순 |
| Weighted RR | 서버 capacity 다름 |
| Least Connections | 연결 수 적은 곳 |
| Least Response Time | 응답 시간 |
| IP Hash / Consistent Hash | 같은 IP → 같은 서버 |
| Random | 랜덤 |
| EWMA | 이동평균 |

---

## 4. 헬스체크

자세히 → [[health-check]]

```
LB → Backend: GET /health
Backend → LB: 200 OK

LB → Backend: GET /health
Backend → LB: (no response / 503)
              ↓
        LB 가 backend 제외
```

### 종류
- **Passive** — 실제 트래픽의 실패 관찰
- **Active** — 주기적 probe

---

## 5. 세션 친화성 (Sticky Session)

### Cookie 기반
```
첫 요청 → LB cookie SETTER
이후 — 같은 backend
```

### IP 기반
```
같은 IP → 같은 backend (consistent hash)
```

### 단점
- 한 서버 죽으면 — 세션 분산
- Auto scaling 어려움
- 권장 — Stateless + 외부 세션 (Redis)

---

## 6. 구현 — 종류

### Hardware
- F5 BIG-IP (전통)
- Citrix NetScaler

### Software
- HAProxy (L4 + L7)
- Nginx (L7 위주)
- Envoy (modern, dynamic)
- Traefik (Kubernetes)
- IPVS (Linux kernel L4)

### Cloud
- AWS ELB (Classic / ALB / NLB / GWLB)
- GCP Cloud Load Balancing
- Azure Load Balancer / App Gateway / Front Door
- Cloudflare Load Balancing

---

## 7. 토폴로지

### Active-Passive
- 한 서버 active, 다른 stand by
- VRRP / Keepalived

### Active-Active
- 모든 서버 동작
- LB 가 분산

### Anycast
- 같은 IP 가 여러 곳
- BGP 가 가까운 곳 라우팅
- CDN / DNS

자세히 → [[../topics/dns-load-balancing-anycast]]

---

## 8. Direct Server Return (DSR)

```
Client → LB: 요청
LB → Backend: 요청 forward
Backend → Client: 응답 (LB 우회)
```

### 효과
- LB 가 응답 처리 X — 부하 ↓
- 대용량 (스트리밍)

### 단점
- TLS 종료 / 헤더 수정 어려움
- L4 만

---

## 9. 모던 — Proxy Protocol

### 문제
- L4 LB → Backend 시 — 클라이언트 IP 손실
- Backend 가 X-Forwarded-For 못 받음

### Proxy Protocol (HAProxy)
```
PROXY TCP4 1.2.3.4 5.6.7.8 12345 80\r\n
<원래 트래픽>
```

→ Backend 가 클라이언트 IP 알 수 있음.

### v1 / v2
- v1 — 텍스트
- v2 — binary (효율)

---

## 10. SSL Offloading vs Passthrough

### SSL Offloading (Termination)
```
Client ─TLS─→ LB ─평문─→ Backend
```

- LB 가 TLS 종료
- Backend 부하 ↓
- 헤더 검사 / 라우팅 가능
- 내부 망 평문 (보안 고려)

### SSL Passthrough
```
Client ─TLS─→ LB ─TLS─→ Backend
```

- LB 가 TLS 그대로 전달
- E2E 암호화
- 헤더 검사 X

### Re-encryption
```
Client ─TLS1─→ LB ─TLS2─→ Backend
```

- LB 가 한 번 복호화 → 재암호화
- 양쪽 검사 + E2E

---

## 11. Failure / Recovery

### 흐름
```
Backend health check failed
        ↓
Mark unhealthy → no new traffic
        ↓
기존 connection — drain (graceful)
        ↓
Recovery 시 — health check pass → re-add
```

### Draining
- 기존 connection 완료까지 기다림
- 새 connection 전송 X
- 시간 한계 (예: 30 초)

---

## 12. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[l4-load-balancing]] | L4 (TCP/UDP) |
| [[l7-load-balancing]] | L7 (HTTP) |
| [[load-balancing-algorithms]] | 알고리즘 |
| [[health-check]] | 헬스체크 |

---

## 13. 학습 자료

- **HAProxy** docs
- **Envoy** docs (Lyft)
- "Scalability Rules" (Abbott)
- AWS / GCP LB docs

---

## 14. 관련

- [[../cdn-proxy/proxy]] — Forward / Reverse proxy
- [[../topics/dns-load-balancing-anycast]] — DNS-level
- [[../tls-ssl/tls-ssl]] — SSL termination
