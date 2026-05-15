---
title: "Load balancer 종류 — ALB / NLB / Classic / GLB"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:44:00+09:00
tags: [devops, networking-ops, load-balancer]
---

# Load balancer 종류 — ALB / NLB / Classic / GLB

**[[networking-ops|↑ networking-ops]]**

---

## 1. AWS ELB 4종

| | Layer | 무엇 | 사용 |
| --- | --- | --- | --- |
| **ALB** (Application LB) | L7 | HTTP/HTTPS, path/host routing | web app |
| **NLB** (Network LB) | L4 | TCP/UDP/TLS, static IP | 고성능, DB proxy |
| **GLB** (Gateway LB) | L3 | 3rd-party 어플라이언스 (방화벽 등) | 보안 도구 통합 |
| **Classic LB** (CLB) | L4+L7 | legacy (사용 X) | (legacy migration) |

→ 신규는 ALB 또는 NLB. **Classic 사용 X**.

---

## 2. ALB (★ web 표준)

```yaml
listeners:
  - port: 443
    protocol: HTTPS
    cert: arn:aws:acm:...
    rules:
      - path: /api/*       → tg-api
      - host: admin.x.com  → tg-admin
      - default            → tg-web
```

기능:
- path / host / header / query 기반 routing
- WebSocket / HTTP/2 / gRPC
- TLS termination (ACM)
- sticky session (cookie 기반)
- WAF 통합 (AWS WAF)
- target type: EC2 / IP / Lambda / containers
- health check (HTTP path)

---

## 3. NLB

```yaml
listeners:
  - port: 5432
    protocol: TCP
    target_group: tg-postgres (TCP)
```

기능:
- ms 수준 latency (L4, kernel-level)
- 정적 IP (per AZ)
- 수백만 connection/s
- PrivateLink endpoint 가능
- TLS passthrough 또는 termination
- UDP / TCP_UDP

→ PG / Redis / Kafka / 일반 TCP 서비스.

---

## 4. GLB

```
[client] → [GLB] → [firewall / IDS / IPS] → [target]
              ↑ transparent (L3, GENEVE tunnel)
```

→ 3rd-party 보안 도구를 traffic flow 에 끼워 넣을 때.

---

## 5. AWS 외

| | 무엇 |
| --- | --- |
| **GCP Cloud Load Balancing** | Global (HTTP, TCP, UDP), Anycast IP |
| **Azure Load Balancer / App Gateway** | LB / WAF + L7 |
| **HAProxy** | OSS L4/L7, on-prem 표준 |
| **F5 BIG-IP** | enterprise hardware |
| **MetalLB** | k8s on bare-metal, LB IP |
| **Cloudflare Spectrum** | TCP/UDP at edge |

---

## 6. health check

### L7 (ALB)

```
Protocol: HTTP
Path: /actuator/health
Healthy threshold: 3
Unhealthy threshold: 3
Timeout: 5s
Interval: 30s
Success codes: 200
```

### L4 (NLB)

```
Protocol: TCP (port open?)
또는 HTTP/HTTPS (TCP+HTTP)
```

→ Spring Boot 에서 `/actuator/health` 이 표준.

---

## 7. sticky session

```yaml
ALB:
  sticky: lb_cookie or app_cookie
  duration: 1h
```

→ JWT / stateless 면 필요 X. session 사용 시만.

---

## 8. cross-zone LB

```
3 AZ, 3 server per AZ
- cross-zone 켬:   각 server 균등 (9 target, equal)
- cross-zone 끔:   AZ 별 균등 (AZ1 의 3 server 가 1/3 traffic)
```

→ ALB default = on, NLB default = off (cross-AZ 비용 ↑).

---

## 9. cost (AWS)

| | 가격 (대략) |
| --- | --- |
| ALB | $0.0225/h + LCU |
| NLB | $0.0225/h + NLCU |
| Classic | $0.025/h + GB |
| GLB | $0.0125/h + GLCU |
| Data 처리 | LCU/NLCU = ~5 connection + traffic 등 복합 |

→ 큰 traffic 시 LCU 가 비용 주범.

---

## 10. ALB vs API Gateway

| | ALB | API Gateway |
| --- | --- | --- |
| L7 routing | ✓ | ✓ |
| TLS | ✓ | ✓ |
| WAF | AWS WAF | 통합 |
| auth | OIDC, Cognito | API key, IAM, Cognito, Lambda Auth |
| rate limit | ✗ | ✓ |
| transformation | ✗ | ✓ |
| cache | ✗ | ✓ |
| cost | 저렴 | 비싸 |

→ 단순 routing = ALB. auth / rate / transform = API Gateway.

---

## 11. nginx / Envoy / HAProxy 와 비교

| | strength | 약점 |
| --- | --- | --- |
| **AWS ALB** | managed, ACM, WAF | AWS lock-in, ALB feature 제한 |
| **nginx** | 익숙, 가벼움 | observability ↓ |
| **Envoy** | gRPC, mesh, observability | 무거움 |
| **HAProxy** | L4 성능 최강 | UI 없음 |

---

## 12. 함정

1. **NLB target type IP — 같은 AZ** — health check 안 됨.
2. **ALB health check path 가 인증 필요** — 항상 401.
3. **cross-zone 끔 + AZ unbalanced** — 한 AZ 만 hot.
4. **WAF 안 함** — bot / SQLi 직접 ALB.
5. **target deregistration delay** — 너무 짧으면 in-flight 끊김 (60s 권장).
6. **NLB 의 client IP preservation** — target 에 PROXY protocol 또는 client IP 보존.

---

## 13. 관련

- [[networking-ops|↑ networking-ops]]
- [[api-gateway]]
- [[../nginx/load-balancing|↗ nginx LB]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
