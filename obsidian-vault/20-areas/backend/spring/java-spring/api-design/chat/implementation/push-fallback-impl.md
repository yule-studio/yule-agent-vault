---
title: "Push fallback 구현 (notification 모듈 호출)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:04:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, push]
---

# Push fallback 구현 (notification 모듈 호출)

**[[implementation|↑ hub]]**

---

## 1. Port

```java
public interface NotificationPort {
    void chatMessageOffline(UserId recipient, ChatMessageNotif payload);
}
```

---

## 2. Adapter (notification 모듈 호출)

```java
@Component
@RequiredArgsConstructor
public class NotificationModuleAdapter implements NotificationPort {

    private final NotificationOutboxService outbox;

    public void chatMessageOffline(UserId recipient, ChatMessageNotif p) {
        outbox.enqueue(NotificationOutboxRow.of(
            UlidId.next(),
            "chat-msg-" + p.messageId(),         // event_id (dedup)
            "CHAT_MESSAGE",
            recipient,
            "CHAT_MESSAGE",                       // template_key
            Map.of(
                "roomName", p.roomName(),
                "senderName", p.senderName(),
                "preview", p.preview(),
                "messageId", p.messageId(),
                "roomId", p.roomId()),
            List.of("FCM", "APNS"),
            NotificationPriority.NORMAL));
    }
}
```

→ notification 모듈의 user preference / quiet hours / debounce 자동 적용.

자세히: [[../../notification/notification|↗ notification]].

---

## 3. 호출 위치

[[message-send-impl#2]] 의 broadcastAndPush 메서드:

```java
for (var m : memberList) {
    if (m.userId().equals(msg.senderId())) continue;
    if (blockFilter.isBlocked(msg.senderId(), m.userId())) continue;
    if (presence.isOnline(m.userId())) continue;
    notif.chatMessageOffline(m.userId(), ...);
}
```

---

## 4. 함정

- online user 도 push → 메시지 + push 중복.
- 사용자 preference 무시 — notification 모듈이 처리.
- 같은 room burst → 같은 사용자 spam → notification 모듈 debounce.

---

## 관련

- [[implementation|↑ hub]]
- [[../design-decisions/push-fallback]]
- [[../../notification/notification|↗ notification recipe]]
