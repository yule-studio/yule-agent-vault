---
title: "Azure Observability — Monitor / Application Insights"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:00:00+09:00
tags:
  - azure
  - observability
  - monitor
  - hub
---

# Azure Observability

**[[../cloud-azure|↑ Azure]]**

---

## 1. 구성

| 영역 | 의미 |
| --- | --- |
| **Azure Monitor** | metric / log 통합 |
| **Log Analytics** | log query (KQL) |
| **Application Insights** | APM (Datadog / NewRelic 동등) |
| **Activity Log** | Azure API audit (AWS CloudTrail) |
| **Diagnostic Settings** | 자원의 log / metric → Log Analytics |
| **Alerts** | metric / log alert |
| **Workbooks** | dashboard |

---

## 2. KQL (Kusto Query Language)

```kql
AppRequests
| where TimeGenerated > ago(1h)
| where ResultCode startswith "5"
| summarize count() by bin(TimeGenerated, 5m), Operation_Name
| render timechart
```

→ SQL-like 강력. Cloud Logging / CloudWatch 보다 강력.

---

## 3. Log Analytics Workspace

모든 log 가 모이는 곳. Subscription 또는 resource 별.

```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = "myapp-law"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}
```

---

## 4. Application Insights

응용 APM — 분산 trace + exception + metric:

```hcl
resource "azurerm_application_insights" "appi" {
  name                = "myapp-appi"
  location            = "koreacentral"
  resource_group_name = azurerm_resource_group.main.name
  application_type    = "web"
  workspace_id        = azurerm_log_analytics_workspace.law.id
}
```

```python
# .NET / Java / Node / Python
# OpenTelemetry 또는 Application Insights SDK
from azure.monitor.opentelemetry import configure_azure_monitor
configure_azure_monitor(connection_string="...")
```

→ Live Metrics / Profiler / Snapshot Debugger.

---

## 5. Diagnostic Settings

```hcl
resource "azurerm_monitor_diagnostic_setting" "vm_to_law" {
  name                       = "to-law"
  target_resource_id         = azurerm_linux_virtual_machine.vm.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.law.id

  enabled_log { category = "Administrative" }
  enabled_log { category = "Security" }
  metric { category = "AllMetrics" }
}
```

→ 자원 활동 log 를 LAW / Storage / Event Hubs 로.

---

## 6. Alerts

```hcl
resource "azurerm_monitor_metric_alert" "cpu" {
  name                = "high-cpu"
  resource_group_name = azurerm_resource_group.main.name
  scopes              = [azurerm_linux_virtual_machine.vm.id]

  criteria {
    metric_namespace = "Microsoft.Compute/virtualMachines"
    metric_name      = "Percentage CPU"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = azurerm_monitor_action_group.alerts.id
  }
}
```

action group = email / SMS / webhook / Logic Apps / Function.

---

## 7. Activity Log = AWS CloudTrail

Azure 의 모든 control plane API audit. 90 일 자동 (LAW 로 보내면 더 길게).

---

## 8. 비용

```
LAW ingest:        $2.30 / GB (Asia)
Retention:         첫 31일 무료, 이후 $0.10 / GB·월
Application Insights: 동일 (LAW 안)
Metric:            대부분 무료
Alert rule:        free / $0.10 per rule·월 (옛)
```

→ ingest 의 정확한 추정. log filtering / sampling 권장.

---

## 9. 외부

- **Grafana** Azure Monitor datasource
- **Datadog Azure integration**
- **Prometheus on AKS** + Managed Prometheus

---

## 10. 관련

- [[../cloud-azure|↑ Azure]]
- [[../../monitoring/monitoring]]
- [[../../../computer-science/operating-system/linux/monitoring|↗ Linux 모니터링]]
