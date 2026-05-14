---
title: "Azure VNet — Virtual Network"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:10:00+09:00
tags:
  - azure
  - network
  - vnet
---

# Azure VNet

**[[network|↑ Network]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

Azure 의 가상 네트워크 — AWS VPC / GCP VPC 동등.
VNet 은 **region 단위** (GCP global VPC 와 다름).

---

## 2. 사용

```bash
az network vnet create -g myapp -n myapp-vnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name app --subnet-prefix 10.0.1.0/24
```

```hcl
resource "azurerm_virtual_network" "vnet" {
  name                = "myapp-vnet"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "app" {
  name                 = "app"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "db" {
  name                 = "db"
  resource_group_name  = azurerm_resource_group.main.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.2.0/24"]
  service_endpoints    = ["Microsoft.Sql", "Microsoft.Storage"]
}
```

---

## 3. NSG (Network Security Group)

AWS SG 동등 — subnet 또는 NIC level.

```hcl
resource "azurerm_network_security_group" "app" {
  name                = "app-nsg"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name

  security_rule {
    name                       = "AllowHttps"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

resource "azurerm_subnet_network_security_group_association" "app" {
  subnet_id                 = azurerm_subnet.app.id
  network_security_group_id = azurerm_network_security_group.app.id
}
```

→ priority (100-4096) — 작을수록 우선. allow + deny default 마지막.

---

## 4. NAT Gateway / Service Endpoint / Private Endpoint

### 4.1 NAT Gateway
private subnet 의 outbound (AWS NAT GW 동등).

### 4.2 Service Endpoint
storage / SQL 등 service 의 backend 가 VNet 안에서만 접근 가능 (공용 endpoint 그대로 IP).

### 4.3 Private Endpoint (권장)
service 가 VNet 안 private IP — 진짜 격리.

---

## 5. VNet Peering / VPN / ExpressRoute

| | |
| --- | --- |
| **VNet Peering** | 두 VNet 1:1 |
| **VPN Gateway** | IPSec on-prem |
| **ExpressRoute** | private 전용 회선 (Direct Connect 동등) |
| **Virtual WAN** | hub-and-spoke (TGW 동등) |

---

## 6. 함정

- NSG = stateful — return 자동 허용
- subnet 의 NSG 와 NIC 의 NSG 동시 — AND
- VNet peering = transitive X
- service endpoint vs private endpoint — private endpoint 권장
- gateway subnet 의 minimum size (/27+)

---

## 7. 관련

- [[network]]
- [[app-gateway]] / [[front-door]]
- [[../security/key-vault]] — private endpoint 자주
