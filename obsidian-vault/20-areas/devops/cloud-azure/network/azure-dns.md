---
title: "Azure DNS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:25:00+09:00
tags:
  - azure
  - network
  - dns
---

# Azure DNS

**[[network|↑ Network]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

매니지드 DNS — Route 53 / Cloud DNS 동등. Public + Private DNS Zone.

---

## 2. 사용

```bash
az network dns zone create -g myapp -n example.com

az network dns record-set a add-record -g myapp -z example.com \
  -n api -a 1.2.3.4

# CNAME
az network dns record-set cname set-record -g myapp -z example.com \
  -n www -c myapp.azurewebsites.net
```

```hcl
resource "azurerm_dns_zone" "main" {
  name                = "example.com"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_dns_a_record" "api" {
  name                = "api"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.main.name
  ttl                 = 300

  target_resource_id = azurerm_public_ip.appgw.id    # alias-like
}

resource "azurerm_dns_cname_record" "www" {
  name                = "www"
  zone_name           = azurerm_dns_zone.main.name
  resource_group_name = azurerm_resource_group.main.name
  ttl                 = 300
  record              = "myapp.azurewebsites.net"
}
```

---

## 3. Private DNS Zone

```hcl
resource "azurerm_private_dns_zone" "internal" {
  name                = "internal.example.com"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_private_dns_zone_virtual_network_link" "link" {
  name                  = "myapp-link"
  resource_group_name   = azurerm_resource_group.main.name
  private_dns_zone_name = azurerm_private_dns_zone.internal.name
  virtual_network_id    = azurerm_virtual_network.vnet.id
}
```

→ VNet 안에서만 resolve.

---

## 4. Traffic Manager — DNS-based 글로벌 routing

```hcl
resource "azurerm_traffic_manager_profile" "tm" {
  name                = "myapp-tm"
  resource_group_name = azurerm_resource_group.main.name
  traffic_routing_method = "Performance"        # Geographic / Weighted / Priority / Subnet

  dns_config {
    relative_name = "myapp-tm"
    ttl           = 60
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = "/health"
  }
}
```

→ DNS 응답 자체를 라우팅. Front Door 와 다름 (anycast LB).

---

## 5. 가격

```
Zone: $0.50 / month / zone
Query: $0.40 / 1M (첫 1B)
```

---

## 6. 함정

- registrar 의 NS 변경 후 전파
- private zone 의 VNet link 누락 → resolve X
- alias record = Public IP / Front Door / Traffic Manager / CDN
- DNSSEC = 옛 (지원 늦음, 신규 zone 지원)

---

## 7. 관련

- [[network]]
- [[front-door]]
- [[../../../computer-science/network/dns/dns|↗ DNS 이론]]
