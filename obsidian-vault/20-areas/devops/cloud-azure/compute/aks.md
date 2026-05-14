---
title: "Azure AKS — Azure Kubernetes Service"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:15:00+09:00
tags:
  - azure
  - compute
  - aks
  - kubernetes
---

# Azure AKS

**[[compute|↑ Compute]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

Azure 의 매니지드 Kubernetes. EKS / GKE 동등. control plane 무료.

---

## 2. 설치

```bash
az aks create \
  --resource-group myapp \
  --name myapp-aks \
  --node-count 3 \
  --node-vm-size Standard_D2as_v5 \
  --enable-managed-identity \
  --enable-cluster-autoscaler \
  --min-count 1 --max-count 10 \
  --network-plugin azure \
  --enable-addons monitoring,azure-keyvault-secrets-provider \
  --workload-identity-enabled \
  --oidc-issuer-enabled \
  --generate-ssh-keys

az aks get-credentials -g myapp -n myapp-aks
kubectl get nodes
```

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name                = "myapp-aks"
  location            = azurerm_resource_group.main.location
  resource_group_name = azurerm_resource_group.main.name
  dns_prefix          = "myapp"

  default_node_pool {
    name       = "default"
    node_count = 3
    vm_size    = "Standard_D2as_v5"
    enable_auto_scaling = true
    min_count = 1
    max_count = 10
  }

  identity { type = "SystemAssigned" }

  oidc_issuer_enabled       = true
  workload_identity_enabled = true

  network_profile {
    network_plugin = "azure"
    network_policy = "calico"
  }
}
```

---

## 3. Workload Identity

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: myapp
  annotations:
    azure.workload.identity/client-id: "<client-id>"
```

→ pod 가 Entra ID 토큰 자동 발급 (GKE Workload Identity / IRSA 동등).

---

## 4. Ingress (Application Gateway 또는 NGINX)

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  ingress_application_gateway {
    gateway_name = "myapp-agic"
    subnet_cidr  = "10.0.10.0/24"
  }
}
```

또는 NGINX Ingress Controller (Helm).

---

## 5. 비용

```
Control plane: 무료 (또는 Uptime SLA $0.10/h = $73/월)
Node: VM 비용 + 관리 디스크 + LB
Free tier: control plane only (best-effort SLA)
```

---

## 6. 함정

- network plugin: **Azure CNI** (직접 IP) vs **kubenet** (NAT). 큰 클러스터 = Azure CNI Overlay.
- IP 소모 — subnet CIDR 큰 / 작은 신중
- workload identity 활성 필수 (옛 pod identity deprecated)
- node 자동 upgrade — release channel 설정

---

## 7. 관련

- [[compute]]
- [[vm]]
- [[container-apps]] — 단순
- [[../../kubernetes/kubernetes]]
