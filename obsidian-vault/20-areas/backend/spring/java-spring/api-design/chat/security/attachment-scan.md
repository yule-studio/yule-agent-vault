---
title: "Attachment scan — virus / malware"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:32:00+09:00
tags: [backend, java-spring, api-design, chat, security, attachment]
---

# Attachment scan — virus / malware

**[[security|↑ hub]]**

---

## 1. 본 vault — 옵션 (F8 이후)

| 단계 | 적용 |
| --- | --- |
| F0~F7 | mime + size 검증 만 |
| F8+ | + ClamAV 비동기 scan |
| F12+ | + AI 기반 malware (옵션) |

---

## 2. 흐름 (F8+)

```mermaid
sequenceDiagram
    participant FE
    participant Server
    participant S3
    participant ClamAV

    FE->>Server: POST /attachments/upload-url
    Server-->>FE: presigned URL
    FE->>S3: PUT
    S3-->>FE: ok
    FE->>Server: POST /messages (attachmentKey=...)
    Server->>S3: read magic byte (mime 검증)
    Server->>DB: messages INSERT (status=SENT but attachment pending)
    Server->>ClamAV: scan request (S3 key)
    par
        Server->>WS: broadcast (preview only)
    end
    Note over ClamAV: 비동기 30초~3분
    ClamAV->>Server: scan result
    alt clean
        Server->>DB: attachment status=READY
        Server->>WS: broadcast "attachment ready"
    else infected
        Server->>S3: DELETE
        Server->>DB: message hidden
        Server->>WS: broadcast "attachment removed"
    end
```

---

## 3. mime 검증 (magic byte)

```java
byte[] head = s3.readBytes(key, 0, 16);
var detector = new Tika();
var detected = detector.detect(head);
if (!allowedMimes.contains(detected)) {
    s3.delete(key);
    throw new InvalidMimeException();
}
```

---

## 4. 함정

1. **FE mime 신뢰** → magic byte 검증.
2. **scan 동기** → 사용자 5분 대기.
3. **scan 결과 broadcast 안 함** → 사용자 못 보임 → 화면 stuck.
4. **infected file 영구 보관** → cleanup.

---

## 관련

- [[security|↑ hub]]
- [[../design-decisions/attachment-strategy]]
