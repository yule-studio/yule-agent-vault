---
title: "FCM Channel 구현 ★ (Firebase Admin SDK)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:46:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, fcm]
---

# FCM Channel 구현 ★ (Firebase Admin SDK)

**[[implementation|↑ hub]]**

---

## 1. 초기화

```java
@Configuration
public class FirebaseConfig {

    @Bean
    public FirebaseApp firebaseApp(@Value("${firebase.service-account-json}") String json)
            throws IOException {
        var credentials = GoogleCredentials.fromStream(
            new ByteArrayInputStream(json.getBytes(UTF_8)));
        var options = FirebaseOptions.builder()
            .setCredentials(credentials)
            .build();
        return FirebaseApp.initializeApp(options);
    }

    @Bean
    public FirebaseMessaging firebaseMessaging(FirebaseApp app) {
        return FirebaseMessaging.getInstance(app);
    }
}
```

→ `firebase.service-account-json` 은 Vault / AWS Secrets Manager 에서 주입.

---

## 2. Channel 구현

```java
@Component
@RequiredArgsConstructor
public class FcmChannel implements NotificationChannel {

    private final FirebaseMessaging messaging;
    private final UserDeviceRepository deviceRepo;
    private final Clock clock;

    public ChannelType type() { return ChannelType.FCM; }

    public DevicePlatform[] supportedPlatforms() {
        return new DevicePlatform[] { DevicePlatform.ANDROID, DevicePlatform.IOS };
    }

    public DeliveryResult send(NotificationPayload payload, UserDevice device) {
        var message = Message.builder()
            .setToken(device.token().value())
            .setNotification(Notification.builder()
                .setTitle(payload.title())
                .setBody(payload.body())
                .setImage(payload.imageUrl())
                .build())
            .putAllData(payload.data())
            .setAndroidConfig(AndroidConfig.builder()
                .setPriority(toAndroidPriority(payload.priority()))
                .setTtl(Duration.ofDays(7).toMillis())
                .build())
            .setApnsConfig(ApnsConfig.builder()
                .setAps(Aps.builder()
                    .setSound("default")
                    .setBadge(1)
                    .build())
                .build())
            .build();

        try {
            var messageId = messaging.send(message);
            return DeliveryResult.success(ChannelType.FCM, messageId, clock.now());

        } catch (FirebaseMessagingException e) {
            return handleError(device, e);
        }
    }

    private DeliveryResult handleError(UserDevice device, FirebaseMessagingException e) {
        var code = e.getMessagingErrorCode();
        switch (code) {
            case UNREGISTERED, INVALID_ARGUMENT -> {
                deviceRepo.markInvalid(device.id(), code.name());
                return DeliveryResult.permanent(ChannelType.FCM,
                    code.name(), e.getMessage(), clock.now());
            }
            case QUOTA_EXCEEDED, INTERNAL, UNAVAILABLE -> {
                return DeliveryResult.transient(ChannelType.FCM,
                    code.name(), e.getMessage(), clock.now());
            }
            case THIRD_PARTY_AUTH_ERROR -> {
                slack.alert("FCM auth error — service account issue", e.getMessage());
                return DeliveryResult.transient(ChannelType.FCM,
                    code.name(), e.getMessage(), clock.now());
            }
            default -> {
                return DeliveryResult.transient(ChannelType.FCM,
                    code != null ? code.name() : "UNKNOWN",
                    e.getMessage(), clock.now());
            }
        }
    }

    private AndroidConfig.Priority toAndroidPriority(NotificationPriority p) {
        return p == NotificationPriority.HIGH || p == NotificationPriority.CRITICAL
            ? AndroidConfig.Priority.HIGH
            : AndroidConfig.Priority.NORMAL;
    }
}
```

---

## 3. 다중 발송 (batch — multi-device user)

```java
// MulticastMessage — 한 사용자의 multi-device 동시 발송
var message = MulticastMessage.builder()
    .addAllTokens(tokens)
    .setNotification(...)
    .build();

var response = messaging.sendEachForMulticast(message);
for (int i = 0; i < response.getResponses().size(); i++) {
    var r = response.getResponses().get(i);
    if (!r.isSuccessful()) {
        var e = r.getException();
        if (e.getMessagingErrorCode() == UNREGISTERED) {
            deviceRepo.markInvalid(devices.get(i).id(), "UNREGISTERED");
        }
    }
}
```

→ 500 token / batch 한도.

---

## 4. 함정

- service account JSON git 노출.
- UNREGISTERED 무시.
- priority HIGH 남발 → Android battery 영향 + FCM rate limit.
- topic 활용 안 함 (group push) → 같은 메시지 N user 매번 발송.

---

## 5. 관련

- [[implementation|↑ hub]]
- [[../security/device-token-rotation]]
- [[../prerequisites#2]]
