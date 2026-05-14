---
title: "OCI OKE — Oracle Kubernetes Engine"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:15:00+09:00
tags:
  - oci
  - compute
  - oke
  - kubernetes
---

# OCI OKE

**[[compute|↑ Compute]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

매니지드 Kubernetes — EKS / GKE / AKS 동등. **control plane 무료** + node 의 ARM Ampere = 가성비 최고.

---

## 2. 설치

```bash
oci ce cluster create \
  --name myapp-oke \
  --compartment-id <ocid> \
  --vcn-id <vcn-ocid> \
  --kubernetes-version v1.29.1 \
  --service-lb-subnet-ids '["<lb-subnet>"]' \
  --kms-key-id <kms-key>

# kubeconfig
oci ce cluster create-kubeconfig --cluster-id <id> --file ~/.kube/config

kubectl get nodes
```

```hcl
resource "oci_containerengine_cluster" "oke" {
  compartment_id     = var.compartment_ocid
  kubernetes_version = "v1.29.1"
  name               = "myapp-oke"
  vcn_id             = oci_core_vcn.vcn.id

  cluster_pod_network_options {
    cni_type = "OCI_VCN_IP_NATIVE"          # pod IP 가 VCN IP
  }

  endpoint_config {
    is_public_ip_enabled = false
    subnet_id            = oci_core_subnet.k8s_api.id
  }

  options {
    service_lb_subnet_ids = [oci_core_subnet.lb.id]
    add_ons { is_kubernetes_dashboard_enabled = false }
  }
}

resource "oci_containerengine_node_pool" "default" {
  cluster_id         = oci_containerengine_cluster.oke.id
  compartment_id     = var.compartment_ocid
  name               = "default"
  node_shape         = "VM.Standard.A1.Flex"
  node_shape_config {
    ocpus         = 2
    memory_in_gbs = 12
  }
  node_source_details {
    image_id    = "ocid1.image..."           # ARM
    source_type = "IMAGE"
  }
  node_config_details {
    placement_configs {
      availability_domain = "..."
      subnet_id           = oci_core_subnet.workers.id
    }
    size = 3
  }
}
```

---

## 3. CNI

| CNI | 의미 |
| --- | --- |
| **Flannel** | overlay (옛) |
| **OCI_VCN_IP_NATIVE** | pod 가 VCN IP — 권장 |

→ VCN-Native CNI = AWS VPC CNI 동등 (pod 가 진짜 VCN IP).

---

## 4. Workload Identity / OCI Service Operator

OCI Service Operator → pod 가 OCI 자원 직접 (S3-호환 / Vault).

또는 instance principal 사용 (worker node 의 권한).

---

## 5. Ingress (LB Service)

`Service type=LoadBalancer` → OCI LB 자동 생성.

```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    oci.oraclecloud.com/load-balancer-type: "lb"     # 또는 "nlb"
spec:
  type: LoadBalancer
  ports: [...]
```

또는 NGINX Ingress + LB Service.

---

## 6. 비용

```
Control plane:     무료 (Basic) — Enhanced $0.10/h
Node:               VM 비용
LB:                 별도
egress:             10 TB 무료/월 (Always Free)
```

→ 가장 저렴한 매니지드 K8s 중 하나.

---

## 7. 함정

- VCN-Native CNI 의 subnet IP 소모 (pod 마다 IP)
- Enhanced cluster = newer features (Workload Identity 등)
- node 자동 upgrade 옵션
- ARM = 이미지 / OCPU 별로 별도

---

## 8. 관련

- [[compute]]
- [[instance]]
- [[../../kubernetes/kubernetes]]
