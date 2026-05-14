---
title: "OCI DNS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:10:00+09:00
tags:
  - oci
  - network
  - dns
---

# OCI DNS

**[[network|↑ Network]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

매니지드 DNS — Route 53 / Cloud DNS 동등. Public Zone + Private Zone + Traffic Steering.

---

## 2. 사용

```bash
oci dns zone create -c <compartment-ocid> --name example.com --zone-type PRIMARY

oci dns record rrset update --zone-name-or-id example.com --domain api.example.com --rtype A \
  --items '[{"domain":"api.example.com","rtype":"A","ttl":300,"rdata":"1.2.3.4"}]'
```

```hcl
resource "oci_dns_zone" "main" {
  compartment_id = var.compartment_ocid
  name           = "example.com"
  zone_type      = "PRIMARY"
}

resource "oci_dns_rrset" "api" {
  zone_name_or_id = oci_dns_zone.main.id
  domain          = "api.example.com"
  rtype           = "A"
  items {
    domain = "api.example.com"
    rtype  = "A"
    ttl    = 300
    rdata  = oci_load_balancer_load_balancer.lb.ip_address_details[0].ip_address
  }
}
```

---

## 3. Private DNS

```hcl
resource "oci_dns_view" "internal" {
  compartment_id = var.compartment_ocid
  display_name   = "internal"
}

resource "oci_dns_zone" "internal" {
  compartment_id = var.compartment_ocid
  name           = "internal.example.com"
  zone_type      = "PRIMARY"
  view_id        = oci_dns_view.internal.id
  scope          = "PRIVATE"
}

# VCN attach
resource "oci_dns_resolver" "vcn" {
  resolver_id = data.oci_core_vcn_dns_resolver_association.vcn.dns_resolver_id
  attached_views {
    view_id = oci_dns_view.internal.id
  }
}
```

---

## 4. Traffic Steering

GTM-like — geo / failover / weight 응답.

```hcl
resource "oci_dns_steering_policy" "geo" {
  compartment_id = var.compartment_ocid
  template       = "ROUTE_BY_GEO"
  display_name   = "geo-routing"

  answers { name = "asia"; rtype = "A"; rdata = "1.2.3.4" }
  answers { name = "us";   rtype = "A"; rdata = "5.6.7.8" }

  rules {
    rule_type = "FILTER"
    cases { case_condition = "query.client.geoKey eq 'KR'"; answer_data { ... } }
  }
}
```

---

## 5. 가격

```
Zone:      $0.50 / month / zone (public)
Query:     $0.40 / 1M (첫 1B)
```

---

## 6. 함정

- registrar NS 변경 후 24h+ 전파
- private zone 의 VCN view attach 누락 → resolve X
- traffic steering = 별도 비용
- Always Free = 자체 DNS resolver 와 별도 (DNS service 는 유료)

---

## 7. 관련

- [[network]]
- [[load-balancer]]
- [[../../../computer-science/network/dns/dns|↗ DNS 이론]]
