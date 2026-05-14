---
title: "Azure Application Gateway"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:15:00+09:00
tags:
  - azure
  - network
  - app-gateway
---

# Azure Application Gateway

**[[network|↑ Network]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

L7 LB + WAF — AWS ALB 동등. URL routing / SSL termination / WebSocket / HTTP/2.

---

## 2. SKU

- **Standard_v2** — autoscale
- **WAF_v2** — + WAF (OWASP rules)

→ 신규 = WAF_v2.

---

## 3. 사용

```hcl
resource "azurerm_application_gateway" "appgw" {
  name                = "myapp-agw"
  resource_group_name = azurerm_resource_group.main.name
  location            = "koreacentral"

  sku {
    name = "WAF_v2"; tier = "WAF_v2"
  }

  autoscale_configuration {
    min_capacity = 2; max_capacity = 10
  }

  gateway_ip_configuration {
    name      = "gw-ipconfig"
    subnet_id = azurerm_subnet.appgw.id
  }

  frontend_ip_configuration {
    name                 = "public"
    public_ip_address_id = azurerm_public_ip.appgw.id
  }

  frontend_port { name = "https"; port = 443 }

  ssl_certificate {
    name                = "cert"
    key_vault_secret_id = azurerm_key_vault_certificate.cert.secret_id
  }

  backend_address_pool { name = "app-pool" }

  backend_http_settings {
    name                  = "app-settings"
    cookie_based_affinity = "Disabled"
    port                  = 8080
    protocol              = "Http"
    request_timeout       = 30
    probe_name            = "health"
  }

  probe {
    name     = "health"
    protocol = "Http"
    path     = "/health"
    interval = 30
    timeout  = 5
    unhealthy_threshold = 3
  }

  http_listener {
    name                           = "https"
    frontend_ip_configuration_name = "public"
    frontend_port_name             = "https"
    protocol                       = "Https"
    ssl_certificate_name           = "cert"
  }

  request_routing_rule {
    name                       = "main"
    priority                   = 100
    rule_type                  = "Basic"
    http_listener_name         = "https"
    backend_address_pool_name  = "app-pool"
    backend_http_settings_name = "app-settings"
  }

  waf_configuration {
    enabled          = true
    firewall_mode    = "Prevention"
    rule_set_type    = "OWASP"
    rule_set_version = "3.2"
  }
}
```

---

## 4. AKS 통합 — AGIC

```hcl
ingress_application_gateway {
  gateway_id = azurerm_application_gateway.appgw.id
}
```

→ Application Gateway Ingress Controller — kubectl 의 Ingress 가 App Gateway 로.

---

## 5. WAF

- OWASP Top 10
- managed rule sets (Microsoft + 자체)
- rate limiting
- geo block

---

## 6. 가격

```
WAF_v2 fixed: $0.443 / 시간 = ~$320/월
+ Capacity Unit per hour
```

비싸지만 ALB + WAF 통합.

---

## 7. 함정

- subnet 의 size (/27+, 모든 인스턴스 IP)
- backend 의 health check (NSG + 응용 path)
- AGIC = AKS pod 의 IP 직접 backend (CNI Azure 필요)
- private 만 = internal mode

---

## 8. 관련

- [[network]]
- [[front-door]] — global
- [[../compute/aks]]
