---
title: "WebPush Channel 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:50:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, webpush]
---

# WebPush Channel 구현

**[[implementation|↑ hub]]**

---

## 1. VAPID 설정

```yaml
webpush:
  vapid-public-key: ${VAPID_PUBLIC}
  vapid-private-key: ${VAPID_PRIVATE}
  subject: mailto:admin@example.com
```

---

## 2. 코드 (nl.martijndwars:web-push)

```java
@Component
@RequiredArgsConstructor
public class WebPushChannel implements NotificationChannel {

    private final PushService pushService;
    private final ObjectMapper mapper;
    private final UserDeviceRepository deviceRepo;
    private final Clock clock;

    public ChannelType type() { return ChannelType.WEBPUSH; }

    public DevicePlatform[] supportedPlatforms() {
        return new DevicePlatform[] { DevicePlatform.WEB, DevicePlatform.DESKTOP };
    }

    public DeliveryResult send(NotificationPayload payload, UserDevice device) {
        // device.token = JSON subscription (endpoint + keys.p256dh + keys.auth)
        var sub = mapper.readValue(device.token().value(), Subscription.class);

        var json = mapper.writeValueAsString(Map.of(
            "title", payload.title(),
            "body", payload.body(),
            "deeplink", payload.deeplink(),
            "icon", "/icon.png"));

        try {
            var notif = new Notification(sub, json);
            var response = pushService.send(notif);

            if (response.getStatusLine().getStatusCode() == 201) {
                return DeliveryResult.success(ChannelType.WEBPUSH,
                    "ok", clock.now());
            }
            if (response.getStatusLine().getStatusCode() == 410) {
                deviceRepo.markInvalid(device.id(), "GONE");
                return DeliveryResult.permanent(ChannelType.WEBPUSH,
                    "410", "subscription gone", clock.now());
            }
            return DeliveryResult.transient(ChannelType.WEBPUSH,
                String.valueOf(response.getStatusLine().getStatusCode()),
                response.getStatusLine().getReasonPhrase(),
                clock.now());

        } catch (Exception e) {
            return DeliveryResult.transient(ChannelType.WEBPUSH, "EX", e.getMessage(), clock.now());
        }
    }
}
```

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../prerequisites#5]]
