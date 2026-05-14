---
title: "OCI Compute Instance"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:10:00+09:00
tags:
  - oci
  - compute
  - instance
---

# OCI Compute Instance

**[[compute|↑ Compute]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

OCI 의 가상 머신. **Flexible shape** (OCPU / RAM 자유) + ARM Ampere 가성비 + Always Free.

---

## 2. Shape

| Shape | 의미 |
| --- | --- |
| **VM.Standard.A1.Flex** | ARM Ampere — Always Free 가능 |
| **VM.Standard.E4/E5.Flex** | AMD EPYC |
| **VM.Standard3.Flex** | Intel |
| **VM.Optimized3.Flex** | high freq Intel |
| **BM.Standard.E5** | Bare Metal |
| **GPU.A10/A100** | NVIDIA GPU |

→ OCPU = 물리 core (hyperthread 2 = vCPU 2). AWS / Azure vCPU 와 다른 단위.

---

## 3. 설치

```hcl
resource "oci_core_instance" "vm" {
  compartment_id      = var.compartment_ocid
  availability_domain = data.oci_identity_availability_domain.ad.name
  display_name        = "myapp-vm"
  shape               = "VM.Standard.A1.Flex"        # ARM, Always Free 가능

  shape_config {
    ocpus         = 2
    memory_in_gbs = 12
  }

  source_details {
    source_type = "image"
    source_id   = "ocid1.image..."                    # Ubuntu 22.04 ARM
  }

  create_vnic_details {
    subnet_id        = oci_core_subnet.app.id
    assign_public_ip = false
  }

  metadata = {
    ssh_authorized_keys = file("~/.ssh/id_ed25519.pub")
    user_data           = base64encode(file("cloud-init.yaml"))
  }
}
```

CLI:
```bash
oci compute instance launch \
  --availability-domain <AD> \
  --compartment-id <ocid> \
  --shape VM.Standard.A1.Flex \
  --shape-config '{"ocpus":2,"memoryInGBs":12}' \
  --image-id <image-ocid> \
  --subnet-id <subnet-ocid>
```

---

## 4. 접속

```bash
ssh opc@<public-ip>           # Oracle Linux
ssh ubuntu@<public-ip>        # Ubuntu

# Bastion service (free)
oci bastion session create ...
```

→ Bastion 권장 (private VM).

---

## 5. 가격 (Seoul)

| Shape | $/시간 | $/월 |
| --- | --- | --- |
| A1.Flex (1 OCPU/6 GB) | **$0** (Always Free up to 4 OCPU) | $0 |
| E4.Flex (1 OCPU/8 GB) | $0.025 | ~$18 |
| E4.Flex (4 OCPU/32 GB) | $0.16 | ~$120 |
| BM.Standard.E5 (192 OCPU) | $4.61 | ~$3360 |

→ AWS / Azure 보다 30-50% 저렴 흔함.

---

## 6. Instance Pool / Autoscaling

```hcl
resource "oci_core_instance_configuration" "config" { ... }
resource "oci_core_instance_pool" "pool" {
  instance_configuration_id = oci_core_instance_configuration.config.id
  size                      = 3
  placement_configurations { ... }
}

resource "oci_autoscaling_auto_scaling_configuration" "asc" {
  policies {
    capacity { min = 1; max = 10; initial = 3 }
    policy_type = "threshold"
    rules { ... }
  }
}
```

---

## 7. Instance Principal

```hcl
resource "oci_identity_dynamic_group" "vm_group" {
  matching_rule = "ALL {instance.compartment.id = '${var.compartment_ocid}'}"
}

resource "oci_identity_policy" "vm_read_buckets" {
  statements = [
    "Allow dynamic-group ${oci_identity_dynamic_group.vm_group.name} to read objects in compartment ${var.compartment_name}"
  ]
}
```

→ VM 안 SDK 자동 인증 (Managed Identity 동등).

---

## 8. 함정

- A1 (ARM) Always Free 한도 4 OCPU / 24 GB — 초과 시 과금
- shape change = stop 필요
- preemptible 도 있음 (Spot 비슷)
- image 의 OCPU / 메모리 minimum
- AD 별 capacity 한계 (ap-seoul-1 = 1 AD)

---

## 9. 관련

- [[compute]]
- [[oke]]
- [[../network/vcn]]
- [[../security/iam]]
