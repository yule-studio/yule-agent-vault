---
title: "Slack Channel 구현 (admin only)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:54:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, slack]
---

# Slack Channel 구현 (admin only)

**[[implementation|↑ hub]]**

---

## 1. 설정

```yaml
slack:
  webhooks:
    payment-alerts: ${SLACK_PAYMENT_WEBHOOK}
    fraud: ${SLACK_FRAUD_WEBHOOK}
    admin-actions: ${SLACK_ADMIN_WEBHOOK}
    infra: ${SLACK_INFRA_WEBHOOK}
```

---

## 2. 코드

```java
@Component
@RequiredArgsConstructor
public class SlackChannel implements NotificationChannel {

    private final RestTemplate http;
    private final SlackProperties props;
    private final ObjectMapper mapper;
    private final Clock clock;

    public ChannelType type() { return ChannelType.SLACK; }

    public DevicePlatform[] supportedPlatforms() {
        return new DevicePlatform[] {};   // admin channel
    }

    public DeliveryResult send(NotificationPayload payload, UserDevice device) {
        // type → channel 매핑
        var notifType = payload.data().get("notificationType");
        var webhookUrl = mapTypeToWebhook(notifType);

        var body = Map.of(
            "text", payload.title(),
            "blocks", buildBlocks(payload));

        try {
            var resp = http.postForEntity(webhookUrl, body, String.class);
            if (resp.getStatusCode().is2xxSuccessful()) {
                return DeliveryResult.success(ChannelType.SLACK,
                    "ok", clock.now());
            }
            return DeliveryResult.transient(ChannelType.SLACK,
                String.valueOf(resp.getStatusCode().value()), "slack-fail", clock.now());

        } catch (Exception e) {
            return DeliveryResult.transient(ChannelType.SLACK,
                "EX", e.getMessage(), clock.now());
        }
    }

    private String mapTypeToWebhook(String type) {
        return switch (type) {
            case "ADMIN_FRAUD_ALERT" -> props.webhooks().get("fraud");
            case "ADMIN_5XX_SPIKE" -> props.webhooks().get("infra");
            case "ADMIN_PAYMENT_ALERT" -> props.webhooks().get("payment-alerts");
            default -> props.webhooks().get("admin-actions");
        };
    }
}
```

---

## 3. 함정

- webhook URL git 노출 → 누구나 알림 발송 가능.
- Slack rate limit (workspace 별 한도) — 너무 많은 알림.
- 사용자 알림 X (admin only).

---

## 4. 관련

- [[implementation|↑ hub]]
- [[../prerequisites#6]]
