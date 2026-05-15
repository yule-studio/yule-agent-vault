---
title: "Networking operations — LB / Service Mesh / API Gateway / CDN"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:40:00+09:00
tags: [area, devops, networking-ops]
---

# Networking operations — LB / Service Mesh / API Gateway / CDN

**[[../devops|↑ devops]]**

---

## 1. 영역

network 의 "사용자 ↔ 서비스" 흐름에 들어가는 모든 구성요소:

```
[User]
   ↓ DNS
[CDN (CloudFront / Cloudflare)]
   ↓
[WAF / DDoS protection]
   ↓
[Public LB (ALB/NLB)]
   ↓
[API Gateway (optional)]
   ↓
[Ingress (k8s) / nginx]
   ↓
[Service Mesh (Istio/Linkerd)]
   ↓
[Application Pod]
   ↓
[Internal LB → DB / Redis / Kafka]
```

→ 모든 레이어는 각자의 책임 + 트레이드오프.

---

## 2. 하위 영역

- [[osi-model]] — L4/L7 차이
- [[load-balancer-types]] — ALB / NLB / classic / GLB
- [[api-gateway]] — Kong / Apigee / AWS API Gateway
- [[service-mesh]] — Istio / Linkerd / Consul
- [[cdn]] — CloudFront / Cloudflare / Fastly
- [[waf]] — Cloudflare / AWS WAF / ModSecurity
- [[dns]] — Route53 / Cloudflare DNS / 권한위임
- [[network-policy]] — k8s NetworkPolicy / Cilium
- [[ssl-tls-ops]] — cert manage / ACM / cert-manager
- [[pitfalls]]

---

## 3. layer 별 무엇

| Layer | 도구 | 무엇 |
| --- | --- | --- |
| **CDN** | CloudFront, Cloudflare | static cache, edge POP |
| **WAF** | AWS WAF, Cloudflare | SQL / XSS / OWASP rule |
| **DDoS** | Shield, Cloudflare | volumetric 차단 |
| **DNS** | Route53, Cloudflare | resolution + LB (failover) |
| **L4 LB** | NLB, HAProxy | TCP/UDP, IP+port |
| **L7 LB** | ALB, nginx, Envoy | HTTP, header/path 기반 |
| **API GW** | Kong, AWS API GW | auth / rate / transform |
| **Ingress (k8s)** | nginx-ingress, Traefik | k8s 내부 진입 |
| **Service Mesh** | Istio, Linkerd | service ↔ service mTLS / observability |

---

## 4. 디자인 패턴

### 4-1. simple

```
User → DNS → ALB → nginx → app
```

→ 소규모, 단순.

### 4-2. CDN + LB

```
User → CDN → ALB → app
              ↓
            cache miss → origin
```

→ 글로벌 / static heavy.

### 4-3. API Gateway

```
User → CDN → ALB → API GW → auth → backend
                        ↓
                      rate limit
                      transform
                      analytics
```

→ MSA / 외부 API.

### 4-4. Service Mesh

```
User → ALB → Ingress → Service Mesh → backend
                            ↓
                          mTLS / retry / circuit break
```

→ MSA / observability 요구.

---

## 5. 학습 순서

1. Day 1 — [[osi-model]] (L4/L7), [[dns]]
2. Day 2 — [[load-balancer-types]] (ALB/NLB), [[ssl-tls-ops]]
3. Day 3 — [[api-gateway]], [[cdn]]
4. Day 4 — [[service-mesh]] (Istio)
5. Day 5 — [[waf]], [[network-policy]]

---

## 6. 관련

- [[../devops|↑ devops]]
- [[../nginx/nginx|↗ nginx]]
- [[../kubernetes/ingress-controllers|↗ k8s ingress]]
- [[../cloud-aws/cloud-aws|↗ AWS]]
