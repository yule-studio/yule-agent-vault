---
title: "notification architecture — Hexagonal + ChannelRouter"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:13:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - architecture
---

# notification architecture — Hexagonal + ChannelRouter

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[notification|↑ hub]]**

---

## 1. 패키지 구조

```
com.example.notification
├── domain/
│   ├── Notification.java
│   ├── NotificationType.java
│   ├── NotificationStatus.java
│   ├── UserDevice.java
│   ├── UserNotificationPreference.java
│   ├── NotificationTemplate.java
│   ├── repository/
│   │   ├── NotificationOutboxRepository.java
│   │   ├── UserDeviceRepository.java
│   │   └── UserPreferenceRepository.java
│   └── port/
│       └── NotificationChannel.java       ← Port (out)
│
├── application/
│   ├── NotificationListenerRegistry.java  ← 도메인별 listener 모음
│   ├── OutboxPublishWorker.java            ★
│   ├── ChannelRouter.java                  ★
│   ├── TemplateRenderer.java
│   └── DeviceManagementService.java
│
├── infrastructure/
│   ├── channel/
│   │   ├── FcmChannel.java                 ← Firebase Admin SDK
│   │   ├── ApnsChannel.java                ← Pushy / HTTP2
│   │   ├── SesEmailChannel.java            ← AWS SDK
│   │   ├── WebPushChannel.java
│   │   ├── SlackChannel.java
│   │   └── InAppChannel.java
│   ├── persistence/jpa/...
│   └── template/
│       └── ThymeleafTemplateRenderer.java
│
├── interfaces/
│   ├── api/                                ← 사용자 endpoints
│   │   ├── DeviceController.java           ← 등록 / 삭제
│   │   ├── PreferenceController.java
│   │   └── InAppNotificationController.java
│   └── admin/
│       └── AdminReplayController.java      ← DLQ replay
│
└── config/
    ├── ChannelRouterConfig.java
    └── ScheduledTasksConfig.java
```

---

## 2. ChannelRouter — 핵심

```java
public interface NotificationChannel {
    ChannelType type();
    DevicePlatform[] supportedPlatforms();
    DeliveryResult send(NotificationPayload payload, UserDevice device);
}
```

```java
@Component
@RequiredArgsConstructor
public class ChannelRouter {

    private final Map<ChannelType, NotificationChannel> byType;
    private final UserDeviceRepository devices;

    public List<DeliveryResult> route(NotificationOutboxRow row) {
        // 1. 채널 결정
        var channelType = decideChannel(row);

        // 2. device 조회
        var userDevices = devices.findActiveByUserId(row.userId(), channelType);

        // 3. 각 device 발송
        var channel = byType.get(channelType);
        return userDevices.stream()
            .map(d -> channel.send(row.toPayload(), d))
            .toList();
    }

    private ChannelType decideChannel(NotificationOutboxRow row) {
        // 1. notification.type 의 default channel
        // 2. user preference 의 channel override
        // 3. 최종 fallback (in-app)
        return switch (row.notificationType()) {
            case PAYMENT_APPROVED, SECURITY_ALERT -> ChannelType.IN_APP;   // 강제 + email 도
            case CHAT_MESSAGE -> ChannelType.FCM;        // app push 우선
            case ADMIN_FRAUD_ALERT -> ChannelType.SLACK;
            default -> ChannelType.IN_APP;
        };
    }
}
```

자세히: [[implementation/channel-router-impl]].

---

## 3. 의존성 흐름

```mermaid
flowchart TB
    Domain[Other domains<br/>signup / board / product / chat] -->|publishEvent| Listener
    Listener[NotificationListener<br/>AFTER_COMMIT] --> Outbox
    Outbox[notification_outbox<br/>DB] --> Worker
    Worker[OutboxPublishWorker<br/>@Scheduled] --> Router
    Router[ChannelRouter] --> Channels
    Channels[NotificationChannel<br/>port] --> Adapters
    Adapters[FCM / APNs / SES / ...]
```

→ 도메인 (signup / board / product) 은 **NotificationChannel port** 만 의존. 채널 추가 시 다른 도메인 영향 X.

---

## 4. 도메인별 listener 패턴

각 도메인 (signup / board / product / chat) 은 자기 영역의 이벤트를 받아 outbox 에 INSERT.

```java
// 예: board
@Component
@RequiredArgsConstructor
class BoardNotificationListener {

    private final NotificationOutboxService outbox;
    private final UserPreferenceService prefs;

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onCommentCreated(CommentCreated ev) {
        // 1. 본인 댓글 skip
        if (ev.authorId().equals(postAuthorId(ev.postId()))) return;

        // 2. preference 확인
        if (!prefs.isEnabled(postAuthorId(ev.postId()), "COMMENT_RECEIVED")) return;

        // 3. outbox INSERT
        outbox.enqueue(NotificationOutboxRow.of(
            UlidId.next(),
            postAuthorId(ev.postId()),
            "COMMENT_RECEIVED",
            Map.of("commenterName", ev.commenterName(), "postId", ev.postId())));
    }
}
```

→ NotificationOutboxService 는 notification 모듈의 public API.

자세히: [[../board/design-decisions/notification-policy|↗ board]].

---

## 5. 모듈 분리 (F12+)

```
F0~F8 (단일 모듈):
  com.example.app
    ├── signup
    ├── board
    ├── product
    └── notification    ← in-process

F12+ (분리):
  com.example.notification-service (별도 Spring Boot app)
    └── Kafka consumer (입력)
    └── ChannelRouter
```

→ 다른 도메인은 Kafka 로 이벤트 발행 → notification-service 가 consumer.

자세히: [[design-decisions/kafka-event-driven]].

---

## 6. 관련

- [[notification|↑ hub]]
- [[transactions]]
- [[domain-model/domain-model]]
- [[implementation/channel-router-impl]]
- [[design-decisions/kafka-event-driven]]
- [[../signup/architecture|↗ signup architecture]]
