---
title: "ChannelRouter 구현 ★"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:44:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, channel]
---

# ChannelRouter 구현 ★

**[[implementation|↑ hub]]**

---

## 1. 코드

```java
@Component
@RequiredArgsConstructor
public class ChannelRouter {

    private final Map<ChannelType, NotificationChannel> channelsByType;
    private final UserDeviceRepository devices;
    private final TemplateRenderer renderer;
    private final ChannelDecider decider;
    private final UserPreferenceService prefs;
    private final Clock clock;

    public Map<ChannelType, DeliveryResult> route(Notification n) {
        // 1. 사용자 preference + type 적용 후 실제 채널 list
        var channels = decider.decide(n.type(), n.userId());

        var results = new HashMap<ChannelType, DeliveryResult>();
        for (var ch : channels) {
            var channel = channelsByType.get(ch);
            if (channel == null) {
                log.warn("channel adapter not registered: {}", ch);
                continue;
            }
            results.put(ch, processChannel(channel, n));
        }
        return results;
    }

    private DeliveryResult processChannel(NotificationChannel channel, Notification n) {
        // 채널 별 device 조회 (FCM/APNS — device list / EMAIL — user email / SLACK — config)
        if (channel.type() == ChannelType.IN_APP) {
            return channel.send(buildPayload(n, null), null);
        }
        if (channel.type() == ChannelType.SLACK) {
            return channel.send(buildPayload(n, null), null);
        }

        // FCM/APNs/WebPush — device 별 발송
        var userDevices = devices.findActiveByUser(n.userId(), channel.type());
        if (userDevices.isEmpty()) {
            return DeliveryResult.transient(channel.type(),
                "no-device", "user has no active device", clock.now());
        }

        // 다중 device 의 결과를 합산 (모두 성공해야 성공)
        boolean anyPermanent = false;
        boolean allSuccess = true;
        for (var d : userDevices) {
            var payload = buildPayload(n, d);
            var result = channel.send(payload, d);
            if (!result.success()) {
                allSuccess = false;
                if (result.permanent()) anyPermanent = true;
            }
        }
        return allSuccess
            ? DeliveryResult.success(channel.type(), "multi-device", clock.now())
            : anyPermanent
                ? DeliveryResult.permanent(channel.type(), "device-fail", "some device fail", clock.now())
                : DeliveryResult.transient(channel.type(), "device-fail", "transient", clock.now());
    }

    private NotificationPayload buildPayload(Notification n, UserDevice device) {
        var template = renderer.render(n, device);
        return new NotificationPayload(
            template.title(),
            template.body(),
            template.deeplink(),
            Map.of("notificationType", n.type().name(), "notificationId", n.id().value()),
            n.priority(),
            null);
    }
}
```

---

## 2. 어댑터 등록 (Spring config)

```java
@Configuration
public class ChannelRouterConfig {

    @Bean
    public Map<ChannelType, NotificationChannel> channelsByType(
            List<NotificationChannel> all) {
        return all.stream()
            .collect(Collectors.toMap(NotificationChannel::type, c -> c));
    }
}
```

→ 새 채널 (예: KakaoTalk Biz) 추가 = 새 `NotificationChannel` Component 등록 만.

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../architecture]]
- [[../domain-model/repository-ports]]
- [[../design-decisions/channel-selection]]
