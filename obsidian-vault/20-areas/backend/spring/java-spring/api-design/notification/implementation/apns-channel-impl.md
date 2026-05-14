---
title: "APNs Channel 구현 ★ (HTTP/2 + JWT)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:48:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, apns]
---

# APNs Channel 구현 ★ (HTTP/2 + JWT)

**[[implementation|↑ hub]]**

---

## 1. Pushy 라이브러리

```java
@Configuration
public class ApnsConfig {

    @Bean
    public ApnsClient apnsClient(ApnsProperties props) throws Exception {
        return new ApnsClientBuilder()
            .setApnsServer(props.isSandbox()
                ? ApnsClientBuilder.DEVELOPMENT_APNS_HOST
                : ApnsClientBuilder.PRODUCTION_APNS_HOST)
            .setSigningKey(ApnsSigningKey.loadFromInputStream(
                new ByteArrayInputStream(props.authKey().getBytes(UTF_8)),
                props.teamId(),
                props.keyId()))
            .build();
    }
}
```

```yaml
apns:
  sandbox: ${APNS_SANDBOX:false}
  team-id: ${APNS_TEAM_ID}
  key-id: ${APNS_KEY_ID}
  auth-key: ${APNS_AUTH_KEY_P8}    # .p8 파일 내용 (KMS / Vault)
  bundle-id: com.example.app
```

---

## 2. Channel 구현

```java
@Component
@RequiredArgsConstructor
public class ApnsChannel implements NotificationChannel {

    private final ApnsClient apnsClient;
    private final ApnsProperties props;
    private final UserDeviceRepository deviceRepo;
    private final ObjectMapper mapper;
    private final Clock clock;

    public ChannelType type() { return ChannelType.APNS; }

    public DevicePlatform[] supportedPlatforms() {
        return new DevicePlatform[] { DevicePlatform.IOS };
    }

    public DeliveryResult send(NotificationPayload payload, UserDevice device) {
        var apsPayload = new SimpleApnsPayloadBuilder()
            .setAlertTitle(payload.title())
            .setAlertBody(payload.body())
            .setBadgeNumber(1)
            .setSound("default")
            .addCustomProperty("deeplink", payload.deeplink())
            .addCustomProperty("notificationType", payload.data().get("notificationType"))
            .build();

        var pushNotification = new SimpleApnsPushNotification(
            device.token().value(),
            props.bundleId(),
            apsPayload,
            null,    // invalidationTime
            mapPriority(payload.priority()),
            PushType.ALERT);

        try {
            var response = apnsClient.sendNotification(pushNotification).get();
            if (response.isAccepted()) {
                return DeliveryResult.success(ChannelType.APNS,
                    response.getApnsId().toString(), clock.now());
            }
            return handleRejection(device, response);

        } catch (Exception e) {
            return DeliveryResult.transient(ChannelType.APNS,
                "EXCEPTION", e.getMessage(), clock.now());
        }
    }

    private DeliveryResult handleRejection(UserDevice device,
                                            PushNotificationResponse<SimpleApnsPushNotification> resp) {
        var reason = resp.getRejectionReason().orElse("UNKNOWN");
        if (reason.contains("BadDeviceToken") || reason.contains("Unregistered")) {
            deviceRepo.markInvalid(device.id(), reason);
            return DeliveryResult.permanent(ChannelType.APNS,
                reason, "device invalid", clock.now());
        }
        if (reason.contains("TooManyRequests") || reason.contains("InternalServerError")) {
            return DeliveryResult.transient(ChannelType.APNS, reason,
                "transient", clock.now());
        }
        return DeliveryResult.transient(ChannelType.APNS, reason, reason, clock.now());
    }

    private DeliveryPriority mapPriority(NotificationPriority p) {
        return p == NotificationPriority.LOW
            ? DeliveryPriority.CONSERVE_POWER
            : DeliveryPriority.IMMEDIATE;
    }
}
```

---

## 3. JWT-based vs Certificate (.p12)

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| **JWT (.p8)** ★ | 영구 key + 다중 app 가능 | JWT 서명 매번 |
| Certificate (.p12) | 단순 | 1년 만료 |

→ Pushy 가 JWT 자동 (1h cache).

---

## 4. 함정

- sandbox / production endpoint 혼동 → 발송 실패.
- bundleId 잘못 → BadDeviceToken.
- HTTP/2 connection pool 누수 — Pushy 자동 관리.
- 410 (Unregistered) 무시.

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../security/device-token-rotation]]
- [[../prerequisites#3]]
