---
title: "chat VOs"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:00:00+09:00
tags: [backend, java-spring, api-design, chat, domain-model, value-object]
---

# chat VOs

**[[domain-model|↑ hub]]**

---

```java
public record RoomId(String value) { ... }
public record MessageId(String value) { ... }
public record SessionId(String value) { ... }
public record DeviceId(String value) { ... }
public record UserId(String value) { ... }     // signup 공유

public record MessageContent(String raw, String sanitized) {
    public static MessageContent of(String raw, HtmlSanitizer sanitizer) {
        return new MessageContent(raw, sanitizer.sanitize(raw));
    }
    public int length() { return raw.length(); }
}

public record Mention(UserId userId, int offset, int length) {}
public record ReplyTo(MessageId messageId, String snippet) {}

public record AttachmentRef(
    String url,
    String thumbnailUrl,
    long fileSize,
    String mimeType,
    Integer width,
    Integer height,
    Integer durationMs
) {}
```

---

## 관련

- [[domain-model|↑ hub]]
- [[message-aggregate]]
