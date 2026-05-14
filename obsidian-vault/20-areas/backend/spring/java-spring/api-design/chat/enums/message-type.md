---
title: "MessageType enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:12:00+09:00
tags: [backend, java-spring, api-design, chat, enum]
---

# MessageType enum

**[[enums|↑ hub]]**

```java
public enum MessageType {
    TEXT,       // 일반 텍스트 (XSS sanitize)
    IMAGE,      // 사진 + thumbnail
    VIDEO,      // 동영상 + first-frame thumbnail
    AUDIO,      // 음성 + waveform
    FILE,       // 일반 파일
    EMOJI,      // 이모티콘 (sticker)
    SYSTEM;     // "OO 님이 들어왔어요" 등 system 메시지
}
```

| Type | content | metadata |
| --- | --- | --- |
| TEXT | sanitized HTML / plain | mentions, replyTo |
| IMAGE | url | width, height, thumbUrl |
| VIDEO | url | duration, thumbUrl |
| AUDIO | url | duration, waveform |
| FILE | url | fileName, mime, size |
| EMOJI | emoji id 또는 url | sticker pack id |
| SYSTEM | i18n key | actor, target |

---

## 관련

- [[enums|↑ hub]]
- [[../database/messages-table]]
- [[../design-decisions/attachment-strategy]]
