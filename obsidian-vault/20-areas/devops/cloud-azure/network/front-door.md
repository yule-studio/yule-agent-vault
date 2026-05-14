---
title: "Azure Front Door"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:20:00+09:00
tags:
  - azure
  - network
  - cdn
  - front-door
---

# Azure Front Door

**[[network|↑ Network]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

**Global L7 LB + CDN** — AWS CloudFront + Global Accelerator + WAF 통합. 전 세계 200+ POP.

---

## 2. Standard vs Premium

| | Standard | Premium |
| --- | --- | --- |
| Global LB | ✅ | ✅ |
| CDN | ✅ | ✅ |
| WAF | basic | OWASP + bot |
| Private Link to origin | X | ✅ |
| Managed rules | 작음 | 풍부 |

---

## 3. 사용

```bash
az afd profile create -g myapp --profile-name myapp-fd --sku Standard_AzureFrontDoor

az afd endpoint create -g myapp --profile-name myapp-fd --endpoint-name myapp-ep
az afd origin-group create -g myapp --profile-name myapp-fd --origin-group-name api \
  --probe-path /health --probe-protocol Http --probe-interval-in-seconds 30

az afd origin create -g myapp --profile-name myapp-fd --origin-group-name api \
  --origin-name api-1 --host-name myapp-agw.example.com --origin-host-header api.example.com

az afd route create -g myapp --profile-name myapp-fd --endpoint-name myapp-ep \
  --route-name main --origin-group api --patterns-to-match "/*" \
  --forwarding-protocol HttpsOnly --supported-protocols Https
```

```hcl
resource "azurerm_cdn_frontdoor_profile" "fd" {
  name                = "myapp-fd"
  resource_group_name = azurerm_resource_group.main.name
  sku_name            = "Standard_AzureFrontDoor"
}

resource "azurerm_cdn_frontdoor_endpoint" "ep" {
  name                     = "myapp"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.fd.id
}
# origin group + origin + route ...
```

---

## 4. Routing

- path 별 origin (`/api/*` → API, `/static/*` → Blob)
- session affinity
- weighted (canary)
- priority (failover)

---

## 5. Caching / Rules

```hcl
# Rule set
- match: PathBeginsWith /static
  action: CacheKeyQueryStringIgnore + CacheExpiration 1 day
- match: HttpVersion Equal 2.0
  action: ...
```

---

## 6. WAF

- managed OWASP rules
- bot manager (Premium)
- rate limit
- geo block

---

## 7. 가격

```
Standard:  $35 / month base + data
Premium:   $165 / month base + data
Outbound:  $0.087/GB (Asia)
Request:   $0.009 / 10K
```

---

## 8. 함정

- propagation 5-10 분
- custom domain + TLS = DNS validation
- origin shielding 옵션
- caching의 query string 정책
- WAF Prevention vs Detection 모드

---

## 9. 관련

- [[network]]
- [[app-gateway]] — regional
- [[../storage/blob-storage]] — origin
- [[../compute/container-apps]] / [[../compute/aks]]
