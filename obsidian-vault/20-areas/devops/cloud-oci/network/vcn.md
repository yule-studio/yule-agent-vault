---
title: "OCI VCN — Virtual Cloud Network"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:00:00+09:00
tags:
  - oci
  - network
  - vcn
---

# OCI VCN

**[[network|↑ Network]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

OCI 의 가상 네트워크 — VPC / VNet 동등. region 단위.

---

## 2. 구성

```
VCN (region)
  ├── Subnet (regional or AD-specific)
  ├── Internet Gateway (IGW)
  ├── NAT Gateway (NAT)
  ├── Service Gateway (SGW)    ← OCI services private
  ├── Dynamic Routing Gateway (DRG) ← VPN / FastConnect / VCN peering
  ├── Local Peering Gateway (LPG) ← same-region peering
  ├── Route Tables
  ├── Security Lists (subnet level)
  └── Network Security Groups (NSG, instance level)
```

---

## 3. 사용

```hcl
resource "oci_core_vcn" "vcn" {
  compartment_id = var.compartment_ocid
  cidr_blocks    = ["10.0.0.0/16"]
  display_name   = "myapp-vcn"
  dns_label      = "myapp"
}

resource "oci_core_internet_gateway" "igw" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.vcn.id
  display_name   = "igw"
}

resource "oci_core_nat_gateway" "nat" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.vcn.id
}

resource "oci_core_subnet" "public" {
  compartment_id    = var.compartment_ocid
  vcn_id            = oci_core_vcn.vcn.id
  cidr_block        = "10.0.1.0/24"
  display_name      = "public"
  prohibit_public_ip_on_vnic = false
  route_table_id    = oci_core_route_table.public.id
  security_list_ids = [oci_core_security_list.public.id]
}

resource "oci_core_subnet" "private" {
  compartment_id    = var.compartment_ocid
  vcn_id            = oci_core_vcn.vcn.id
  cidr_block        = "10.0.2.0/24"
  prohibit_public_ip_on_vnic = true                 # private only
  route_table_id    = oci_core_route_table.private.id
}
```

---

## 4. Security List vs NSG

| | Security List | NSG (Network Security Group) |
| --- | --- | --- |
| 범위 | subnet 의 모든 VM | 명시적 인스턴스 / VNIC |
| Source | CIDR | CIDR + NSG (그룹 to 그룹) |
| 권장 | 폐 / 단순 | 권장 (AWS SG 와 비슷) |

```hcl
resource "oci_core_network_security_group" "app" {
  compartment_id = var.compartment_ocid
  vcn_id         = oci_core_vcn.vcn.id
}

resource "oci_core_network_security_group_security_rule" "ssh_from_bastion" {
  network_security_group_id = oci_core_network_security_group.app.id
  direction                 = "INGRESS"
  protocol                  = "6"            # TCP
  source                    = oci_core_network_security_group.bastion.id
  source_type               = "NETWORK_SECURITY_GROUP"
  tcp_options { destination_port_range { min = 22; max = 22 } }
}
```

---

## 5. Service Gateway / NAT / IGW

- **IGW** = public 양방향 (public subnet 만 사용)
- **NAT** = private subnet 의 outbound 만
- **SGW** = OCI 자원 (Object Storage 등) private 접근 — 자체 region 안

→ private VM 이 Object Storage 사용 = SGW (NAT 불필요, 빠름).

---

## 6. Peering

| | |
| --- | --- |
| **LPG** | 같은 region 의 VCN ↔ VCN |
| **DRG + RPC** | cross-region peering |
| **DRG + VPN** | on-prem |
| **DRG + FastConnect** | 전용 회선 |

---

## 7. 함정

- subnet 의 prohibit_public_ip_on_vnic — public 차단
- security list AND NSG = 둘 다 통과
- DRG attachment 별로 권한 / 라우팅
- VCN CIDR 변경 불가 — 처음 설계 신중
- transit = DRG 가 hub

---

## 8. 관련

- [[network]]
- [[load-balancer]]
- [[../security/iam]]
