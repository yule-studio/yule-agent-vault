---
title: "OCI Load Balancer"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:05:00+09:00
tags:
  - oci
  - network
  - load-balancer
---

# OCI Load Balancer

**[[network|↑ Network]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

OCI 의 LB — L4 NLB + L7 LB (HTTP/HTTPS) 두 종류.

| | LB (Flexible) | NLB |
| --- | --- | --- |
| Layer | L7 | L4 |
| TLS termination | ✅ | passthrough |
| WebSocket | ✅ | ✅ |
| Source IP 보존 | X | ✅ |
| 비용 | 비쌈 | 매우 쌈 |
| Always Free | LB 10Mbps × 1 | NLB 무료 |

---

## 2. 사용 — L7 LB

```hcl
resource "oci_load_balancer_load_balancer" "lb" {
  compartment_id = var.compartment_ocid
  display_name   = "myapp-lb"
  shape          = "flexible"
  shape_details {
    minimum_bandwidth_in_mbps = 10
    maximum_bandwidth_in_mbps = 100
  }
  subnet_ids = [oci_core_subnet.lb.id]
}

resource "oci_load_balancer_backend_set" "app" {
  load_balancer_id = oci_load_balancer_load_balancer.lb.id
  name             = "app-bs"
  policy           = "ROUND_ROBIN"
  health_checker {
    protocol = "HTTP"
    url_path = "/health"
    port     = 8080
  }
}

resource "oci_load_balancer_backend" "b1" {
  load_balancer_id = oci_load_balancer_load_balancer.lb.id
  backendset_name  = oci_load_balancer_backend_set.app.name
  ip_address       = oci_core_instance.vm.private_ip
  port             = 8080
}

resource "oci_load_balancer_listener" "https" {
  load_balancer_id         = oci_load_balancer_load_balancer.lb.id
  name                     = "https"
  port                     = 443
  protocol                 = "HTTP"
  default_backend_set_name = oci_load_balancer_backend_set.app.name

  ssl_configuration {
    certificate_name        = oci_load_balancer_certificate.cert.certificate_name
    verify_peer_certificate = false
  }
}
```

---

## 3. NLB

```hcl
resource "oci_network_load_balancer_network_load_balancer" "nlb" {
  compartment_id = var.compartment_ocid
  display_name   = "myapp-nlb"
  subnet_id      = oci_core_subnet.public.id
  is_private     = false
  is_preserve_source_destination = false
}
```

→ 매우 빠름, 거의 무료.

---

## 4. Health Check

URL path + TCP / HTTP / HTTPS + interval + retry + statusCode.

→ unhealthy = 자동 제외.

---

## 5. 가격

```
LB (10 Mbps min):     $0.0259 / 시간 (Always Free 1 개)
LB (100 Mbps):        ~$0.10 / 시간
NLB:                  무료 (data only)
```

---

## 6. 함정

- LB / NLB subnet 의 size (/27+)
- SSL 의 cipher / TLS 버전 정책 = 별도 (`SSLCipherSuite`)
- WebSocket = idle timeout 길게 설정
- source IP 보존 = NLB 또는 X-Forwarded-For
- backend 의 NSG = LB subnet 의 IP 허용

---

## 7. 관련

- [[network]]
- [[vcn]]
- [[../compute/instance]]
