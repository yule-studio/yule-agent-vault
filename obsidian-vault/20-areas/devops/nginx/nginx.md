---
title: "nginx — reverse proxy / LB / TLS ★"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:05:00+09:00
tags: [area, devops, nginx]
---

# nginx — reverse proxy / LB / TLS ★

**[[../devops|↑ devops]]**

---

## 1. 어디에 쓰나

| 역할 | 무엇 |
| --- | --- |
| **reverse proxy** | client ↔ backend (Spring/Node) |
| **static file** | HTML / CSS / JS / 이미지 (Apache 대체) |
| **load balancer** | upstream {} 으로 분배 |
| **TLS termination** | HTTPS → HTTP 변환 |
| **API gateway** | rate-limit / auth / routing |
| **cache** | proxy_cache (CDN 류) |

→ k8s 환경에선 Ingress Controller 로도 사용.

---

## 2. 대안

| | 장단 |
| --- | --- |
| **nginx** | 가장 표준, OSS / 상용 (Nginx Plus) |
| **Apache httpd** | 기능 풍부, .htaccess (legacy) |
| **HAProxy** | LB 전문 (TCP/HTTP) |
| **Caddy** | auto-HTTPS (Let's Encrypt) |
| **Envoy** | service mesh / gRPC / 현대적 |
| **Traefik** | k8s 친화, 자동 service discovery |

→ **일반 reverse proxy = nginx**, **k8s = nginx-ingress / Traefik**, **service mesh = Envoy/Istio**.

---

## 3. 하위 영역

- [[concepts]] — master/worker, event loop
- [[config-structure]] — http/server/location/upstream
- [[reverse-proxy]] — proxy_pass / header
- [[load-balancing]] — upstream + 알고리즘
- [[tls-https]] — certbot + Let's Encrypt
- [[caching]] — proxy_cache + cache key
- [[rate-limiting]] — limit_req / limit_conn
- [[security]] — header / WAF
- [[performance-tuning]] — worker / keepalive / buffer
- [[pitfalls]]
- [[practice/practice|practice]] — Spring + nginx 통합 lab

---

## 4. 학습 순서

1. Day 1: concepts + config-structure
2. Day 2: reverse-proxy + tls-https
3. Day 3: load-balancing + rate-limiting
4. Day 4: caching + performance-tuning
5. Day 5: practice 실습

---

## 5. 표준 architecture

```
[Internet]
    ↓ HTTPS:443
[ALB / CDN]
    ↓
[nginx (TLS termination)]
    ├─ static (HTML/JS/CSS) — 직접 응답
    └─ /api/* → upstream [spring1, spring2, spring3]
                   ↓ HTTP keepalive
                [Spring Boot] × N
```

---

## 6. 관련

- [[../devops|↑ devops]]
- [[../linux/networking-basics|↗ linux networking]]
- [[../kubernetes/ingress-controllers|↗ k8s ingress]]
