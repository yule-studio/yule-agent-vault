---
title: "Azure VM"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:10:00+09:00
tags:
  - azure
  - compute
  - vm
---

# Azure Virtual Machine

**[[compute|↑ Compute]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

Azure 의 가상 머신 — AWS EC2 / GCP GCE 동등. **Hybrid Benefit** (Windows / SQL Server 라이선스 재사용) 가 강점.

---

## 2. Size

| Family | 용도 |
| --- | --- |
| **B** | burstable (작은) |
| **D** | 범용 |
| **F** | CPU |
| **E** | RAM |
| **L** | NVMe local SSD |
| **N** | GPU |
| **H** | HPC |
| **D / E v5 / v6** | 최신 |

→ 일반 = D2as_v5 / D4as_v5 (AMD). 옛 = Dv3.

---

## 3. 설치

```bash
az group create -n myapp -l koreacentral

az vm create \
  --resource-group myapp \
  --name myapp-vm \
  --image Ubuntu2204 \
  --size Standard_D2as_v5 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name myapp-vnet \
  --subnet private \
  --public-ip-address ""              # private 만
```

```hcl
resource "azurerm_linux_virtual_machine" "vm" {
  name                  = "myapp"
  resource_group_name   = azurerm_resource_group.main.name
  location              = "koreacentral"
  size                  = "Standard_D2as_v5"
  admin_username        = "azureuser"
  network_interface_ids = [azurerm_network_interface.main.id]

  admin_ssh_key {
    username   = "azureuser"
    public_key = file("~/.ssh/id_ed25519.pub")
  }

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Premium_LRS"
    disk_size_gb         = 30
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  identity { type = "SystemAssigned" }      # Managed Identity
}
```

---

## 4. 접속

```bash
# SSH (public IP)
ssh azureuser@<ip>

# Bastion (방화벽 22 안 열어도)
az network bastion ssh --name myapp-bastion --resource-group myapp --target-resource-id ... --auth-type ssh-key

# Run command (CLI 만)
az vm run-command invoke -g myapp -n myapp-vm --command-id RunShellScript --scripts "uptime"
```

→ Bastion / Just-In-Time access 권장.

---

## 5. 가격 (Korea Central)

| Size | $/시간 | $/월 |
| --- | --- | --- |
| B1s (1 vCPU/1GB) | $0.0124 | ~$9 |
| D2as_v5 (2/8GB) | $0.107 | ~$78 |
| D4as_v5 (4/16GB) | $0.213 | ~$155 |
| E4as_v5 (4/32GB RAM) | $0.275 | ~$200 |

+ disk + bandwidth.

---

## 6. VMSS (Virtual Machine Scale Set)

```hcl
resource "azurerm_linux_virtual_machine_scale_set" "app" {
  name                = "myapp-vmss"
  resource_group_name = azurerm_resource_group.main.name
  location            = "koreacentral"
  sku                 = "Standard_D2as_v5"
  instances           = 3

  autoscale ...
}
```

→ AWS Auto Scaling 동등.

---

## 7. Managed Identity (권장)

```hcl
identity { type = "SystemAssigned" }

resource "azurerm_role_assignment" "blob_read" {
  scope                = azurerm_storage_account.main.id
  role_definition_name = "Storage Blob Data Reader"
  principal_id         = azurerm_linux_virtual_machine.vm.identity[0].principal_id
}
```

→ VM 안 SDK 가 자동 인증. key X.

---

## 8. 함정

- public IP 직노출 → Bastion / JIT
- managed disk type — Premium SSD / Standard SSD / Standard HDD
- Spot VM = 강제 종료
- Hybrid Benefit = Windows / SQL Server 라이선스 보유 시 큰 절약
- decommission 전에 OS 안 backup

---

## 9. 관련

- [[compute]]
- [[aks]] / [[container-apps]]
- [[../network/vnet]]
- [[../security/key-vault]]
