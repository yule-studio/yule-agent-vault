---
title: "GCP Cloud Load Balancing"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:10:00+09:00
tags:
  - gcp
  - network
  - load-balancer
---

# GCP Cloud Load Balancing

**[[network|↑ Network]]** · **[[../cloud-gcp|↑↑ GCP]]**

---

## 1. 한 줄

GCP 의 **글로벌 LB** — 단일 anycast IP 로 전 세계 라우팅. 글로벌 latency 자동 최적.

---

## 2. 종류

| Type | OSI | 글로벌 |
| --- | --- | --- |
| **Global External HTTPS** | L7 | ✅ |
| **Regional External HTTPS** | L7 | regional |
| **Global External TCP/SSL Proxy** | L4 proxy | ✅ |
| **Network LB (passthrough)** | L4 | regional |
| **Internal HTTPS** | L7 | regional |
| **Internal TCP/UDP** | L4 | regional |

→ 대부분 **Global External HTTPS** (=AWS ALB + CloudFront 일부 통합).

---

## 3. 글로벌 LB 의 특징

```
사용자 → 가까운 GCP edge (1개 anycast IP)
         → Google backbone
         → 가까운 healthy backend
```

- 단일 anycast IP — 한국 / 미국 / 유럽 어디서나 같은 IP
- Premium Tier (Google backbone) 가 default
- HTTP/3 (QUIC) 지원
- WAF (Cloud Armor)

---

## 4. 설정

```hcl
resource "google_compute_global_address" "lb" {
  name = "myapp-lb-ip"
}

resource "google_compute_managed_ssl_certificate" "cert" {
  name = "myapp-cert"
  managed { domains = ["api.example.com"] }
}

resource "google_compute_health_check" "http" {
  name = "myapp-hc"
  http_health_check {
    port = 8080
    request_path = "/health"
  }
}

resource "google_compute_backend_service" "app" {
  name          = "myapp-bs"
  protocol      = "HTTP"
  health_checks = [google_compute_health_check.http.id]
  load_balancing_scheme = "EXTERNAL_MANAGED"

  backend {
    group = google_compute_region_network_endpoint_group.app.id
  }
}

resource "google_compute_url_map" "lb" {
  name            = "myapp-url"
  default_service = google_compute_backend_service.app.id
}

resource "google_compute_target_https_proxy" "lb" {
  name             = "myapp-https-proxy"
  url_map          = google_compute_url_map.lb.id
  ssl_certificates = [google_compute_managed_ssl_certificate.cert.id]
}

resource "google_compute_global_forwarding_rule" "lb" {
  name        = "myapp-fwd"
  ip_address  = google_compute_global_address.lb.address
  port_range  = "443"
  target      = google_compute_target_https_proxy.lb.id
}
```

→ Cloud DNS 의 A record 를 anycast IP 로.

---

## 5. Cloud Armor (WAF)

```hcl
resource "google_compute_security_policy" "armor" {
  name = "myapp-armor"

  rule {
    priority = 1000
    action   = "deny(403)"
    match {
      expr { expression = "evaluatePreconfiguredExpr('sqli-stable')" }
    }
  }
  rule {
    priority = 2000
    action   = "rate_based_ban"
    rate_limit_options {
      conform_action = "allow"
      exceed_action  = "deny(429)"
      rate_limit_threshold { count = 100; interval_sec = 60 }
    }
    match { versioned_expr = "SRC_IPS_V1"; config { src_ip_ranges = ["*"] } }
  }
}
```

→ SQL injection / XSS / rate limit / 지역 차단.

---

## 6. Identity-Aware Proxy (IAP)

```hcl
backend {
  iap {
    oauth2_client_id     = ...
    oauth2_client_secret = ...
  }
}
```

→ Google login 강제 — VPN 없이 internal app.

---

## 7. 비용

```
Forwarding rule: $0.025 / 시간 + LCU (요청 / 데이터)
Premium Tier:    glob backbone (default)
Standard Tier:   더 싸지만 region binding
```

---

## 8. 함정

- managed cert = DNS validation 후 ~ 30분-2시간
- backend health check 의 firewall (`35.191.0.0/16` IAP / health check range)
- Standard Tier 의 함정 — global anycast 잃음
- TLS policy = TLS 1.0 X

---

## 9. 관련

- [[network]]
- [[cloud-cdn]] — 같은 LB 위에
- [[../compute/cloud-run]] / [[../compute/gke]] — backend
