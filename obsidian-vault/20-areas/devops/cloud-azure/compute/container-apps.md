---
title: "Azure Container Apps"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:20:00+09:00
tags:
  - azure
  - compute
  - container-apps
  - serverless
---

# Azure Container Apps

**[[compute|↑ Compute]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

**Serverless container** — Kubernetes (KEDA + Dapr) 위에 단순한 인터페이스. Cloud Run / ECS Fargate 동등.

---

## 2. 사용

```bash
az containerapp create \
  --name myapp \
  --resource-group myapp \
  --environment myapp-env \
  --image mycr.azurecr.io/myapp:v1 \
  --target-port 8080 \
  --ingress external \
  --cpu 0.5 --memory 1Gi \
  --min-replicas 0 --max-replicas 10
```

```hcl
resource "azurerm_container_app" "myapp" {
  name                         = "myapp"
  container_app_environment_id = azurerm_container_app_environment.env.id
  resource_group_name          = azurerm_resource_group.main.name
  revision_mode                = "Single"

  template {
    container {
      name   = "app"
      image  = "mycr.azurecr.io/myapp:v1"
      cpu    = 0.5
      memory = "1Gi"

      env {
        name  = "DB_HOST"
        value = "..."
      }
    }
    min_replicas = 0
    max_replicas = 10

    http_scale_rule {
      name = "http"
      concurrent_requests = "50"
    }
  }

  ingress {
    external_enabled = true
    target_port      = 8080
    traffic_weight {
      latest_revision = true
      percentage      = 100
    }
  }
}
```

→ scale-to-zero 가능. KEDA 기반 스케일러 (HTTP / Queue / cron / custom).

---

## 3. Revision / Traffic split

```hcl
revision_mode = "Multiple"

traffic_weight {
  revision_suffix = "v1"; percentage = 90
}
traffic_weight {
  revision_suffix = "v2"; percentage = 10
}
```

→ canary / blue-green.

---

## 4. Dapr 통합

Container Apps Environment 에 Dapr 활성 → service discovery / state / pub-sub / secret store.

---

## 5. 가격

```
vCPU: $0.000024 / vCPU·s
Memory: $0.000003 / GB·s
Request: $0.40 / 1M
Free: 180K vCPU·s + 360K GB·s + 2M request / month
```

→ Cloud Run 와 비슷.

---

## 6. 함정

- min_replicas=0 의 cold start
- ingress external = public — 명시
- secret 은 별도 (Key Vault 또는 inline)
- managed identity → ACR pull

---

## 7. 관련

- [[compute]]
- [[aks]] — 본격 K8s
- [[functions]] — 함수
- [[../security/key-vault]]
