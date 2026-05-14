---
title: "Azure Event Grid"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:55:00+09:00
tags:
  - azure
  - messaging
  - event-grid
---

# Azure Event Grid

**[[messaging|↑ Messaging]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

**Event routing** — Azure 자원의 이벤트 (Storage Blob 생성, SQL 변경 등) → subscriber. AWS EventBridge 동등.

---

## 2. 구조

```
Source (publisher) → Topic → Subscription → Handler
                                     filter
```

- **System Topic** — Azure 자원 자동 (Storage, Resource Group, ...)
- **Custom Topic** — 응용이 publish
- **Domain** — 다중 tenant 의 topic 묶음
- **Partner Topic** — 외부 SaaS (Auth0 등)

---

## 3. 사용

```bash
# Custom topic
az eventgrid topic create -g myapp -n myapp-events -l koreacentral

# Subscription — Function 으로
az eventgrid event-subscription create \
  --source-resource-id "/subscriptions/.../topics/myapp-events" \
  --name email-handler \
  --endpoint <function-url>
```

```hcl
resource "azurerm_eventgrid_topic" "events" {
  name                = "myapp-events"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name
}

resource "azurerm_eventgrid_event_subscription" "email" {
  name  = "email"
  scope = azurerm_eventgrid_topic.events.id

  webhook_endpoint {
    url = azurerm_function_app.email.default_hostname
  }

  retry_policy {
    max_delivery_attempts = 30
    event_time_to_live    = 1440
  }

  dead_letter_identity {
    type = "SystemAssigned"
  }
  storage_blob_dead_letter_destination {
    storage_account_id          = azurerm_storage_account.main.id
    storage_blob_container_name = azurerm_storage_container.deadletter.name
  }

  subject_filter {
    subject_begins_with = "/blobServices/default/containers/uploads/"
  }
}
```

---

## 4. Event Schema

CloudEvents 1.0 표준 또는 Azure Event Grid schema:
```json
{
  "id": "...",
  "subject": "/blobServices/default/containers/uploads/blobs/file.jpg",
  "data": { ... },
  "eventType": "Microsoft.Storage.BlobCreated",
  "dataVersion": "2",
  "metadataVersion": "1",
  "eventTime": "2026-05-15T03:00:00Z",
  "topic": "/subscriptions/.../storageAccounts/myapp"
}
```

---

## 5. 가격

```
$0.60 / 1M operation
Free: 100K operation / month
```

→ 매우 저렴.

---

## 6. 함정

- delivery at-least-once
- max delivery attempts (30 default)
- DLQ = blob 으로 dead letter
- webhook validation handshake (응답 200 필수)
- filter 의 SQL 비슷한 limit (5 개)

---

## 7. 관련

- [[messaging]]
- [[service-bus]] — 차이
- [[../compute/functions]] — subscriber
- [[../storage/blob-storage]] — system event
