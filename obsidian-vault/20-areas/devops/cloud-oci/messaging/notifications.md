---
title: "OCI Notifications"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:40:00+09:00
tags:
  - oci
  - messaging
  - notifications
  - pubsub
---

# OCI Notifications

**[[messaging|↑ Messaging]]** · **[[../cloud-oci|↑↑ OCI]]**

---

## 1. 한 줄

Topic-based pub/sub — AWS SNS 동등. email / SMS / HTTPS webhook / Function / Slack.

---

## 2. 사용

```hcl
resource "oci_ons_notification_topic" "alerts" {
  compartment_id = var.compartment_ocid
  name           = "alerts"
}

resource "oci_ons_subscription" "email" {
  compartment_id = var.compartment_ocid
  topic_id       = oci_ons_notification_topic.alerts.id
  protocol       = "EMAIL"
  endpoint       = "ops@example.com"
}

resource "oci_ons_subscription" "slack" {
  compartment_id = var.compartment_ocid
  topic_id       = oci_ons_notification_topic.alerts.id
  protocol       = "HTTPS"
  endpoint       = "https://hooks.slack.com/services/..."
}

resource "oci_ons_subscription" "fn" {
  compartment_id = var.compartment_ocid
  topic_id       = oci_ons_notification_topic.alerts.id
  protocol       = "ORACLE_FUNCTIONS"
  endpoint       = oci_functions_function.handler.id
}
```

---

## 3. Protocol

| Protocol | 의미 |
| --- | --- |
| EMAIL | email |
| SMS | SMS (지역별 가능) |
| HTTPS | webhook (POST) |
| ORACLE_FUNCTIONS | Function 호출 |
| PAGERDUTY | PagerDuty 직통 |
| SLACK | Slack incoming webhook (HTTPS 의 alias) |

---

## 4. Publish

```bash
oci ons message publish --topic-id <id> --title "ALERT" --body "Disk full on instance X"
```

```python
import oci
signer = oci.auth.signers.InstancePrincipalsSecurityTokenSigner()
client = oci.ons.NotificationDataPlaneClient(config={}, signer=signer)
client.publish_message(topic_id="...", message_details=oci.ons.models.MessageDetails(
    title="ALERT", body="..."
))
```

---

## 5. Monitoring → Notifications 통합

Monitoring alarm 이 topic 으로 publish. → 한 곳에서 email / slack / pager 분기.

---

## 6. 가격

```
Free:    1M delivery / month (email / HTTPS)
After:   $0.50 / 1M
SMS:     국가별 ($0.0075 / message US)
```

---

## 7. 함정

- email subscribe 후 confirm 클릭 필요
- HTTPS webhook = 200 응답 필수 (재시도)
- ordering 보장 X (best-effort)
- payload max 64 KB
- SMS = phone number 등록 (sandbox)

---

## 8. 관련

- [[messaging]]
- [[streaming]]
- [[../observability/observability]] — alarm subscriber
