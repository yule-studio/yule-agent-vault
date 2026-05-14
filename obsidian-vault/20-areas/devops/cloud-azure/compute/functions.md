---
title: "Azure Functions"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:25:00+09:00
tags:
  - azure
  - compute
  - functions
  - serverless
---

# Azure Functions

**[[compute|↑ Compute]]** · **[[../cloud-azure|↑↑ Azure]]**

---

## 1. 한 줄

Serverless function — AWS Lambda 동등. C# / Node / Python / Java / PowerShell.

---

## 2. Hosting Plans

| Plan | 의미 |
| --- | --- |
| **Consumption** | scale-to-zero, pay per execution |
| **Premium** | pre-warmed, VNet, longer timeout |
| **Dedicated (App Service)** | 항상 켜둠 |

---

## 3. Trigger

- HTTP
- Timer (cron)
- Blob Storage / Queue Storage
- Service Bus / Event Grid / Event Hubs
- Cosmos DB Change Feed
- ...

---

## 4. 사용

```bash
func init MyFunc --python
cd MyFunc
func new --name HttpExample --template "HTTP trigger"

# 로컬 실행
func start

# 배포
az functionapp create \
  --resource-group myapp \
  --name myapp-func \
  --consumption-plan-location koreacentral \
  --runtime python --runtime-version 3.11 \
  --functions-version 4 \
  --storage-account myappstorage

func azure functionapp publish myapp-func
```

```python
import azure.functions as func
app = func.FunctionApp()

@app.route(route="hello")
def hello(req: func.HttpRequest) -> func.HttpResponse:
    return func.HttpResponse("Hello!")

@app.timer_trigger(schedule="0 */5 * * * *", arg_name="t")
def cron(t: func.TimerRequest):
    print("cron")

@app.blob_trigger(arg_name="blob", path="uploads/{name}", connection="AzureWebJobsStorage")
def on_blob(blob: func.InputStream):
    print(f"blob: {blob.name}")
```

---

## 5. 가격

```
Consumption:
  $0.20 / 1M execution
  $0.000016 / GB·s
  Free: 1M execution + 400K GB·s / month
Premium: pre-warmed instance 비용
```

---

## 6. Durable Functions

상태 / 워크플로우 (Step Functions 동등):

```python
@app.orchestration_trigger(context_name="ctx")
def order_workflow(ctx):
    x = yield ctx.call_activity("ChargePayment", order)
    y = yield ctx.call_activity("SendEmail", order)
    return [x, y]
```

---

## 7. 함정

- cold start (Consumption)
- 5분 timeout (Consumption) / 60분 (Premium)
- 응용 시작 = host.json + local.settings.json
- VNet 통합 = Premium 만 (Consumption X)

---

## 8. 관련

- [[compute]]
- [[container-apps]] — container 기반
- [[../messaging/service-bus]] — trigger
