---
title: "Attachment 구현 (S3 presigned + thumbnail)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:58:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, attachment]
---

# Attachment 구현 (S3 presigned + thumbnail)

**[[implementation|↑ hub]]**

---

## 1. Presigned URL endpoint

```java
@PostMapping("/api/v1/attachments/upload-url")
public ResponseEntity<UploadUrlResponse> getUploadUrl(
        @RequestBody UploadUrlRequest req,
        @AuthenticationPrincipal AuthUser auth) {

    require(allowedMimes.contains(req.mimeType()));
    require(req.size() <= sizeLimitFor(req.mimeType()));

    var key = "chat/%s/%s/%s".formatted(
        auth.userId().value(),
        LocalDate.now(),
        UlidId.next());

    var url = s3.presignedPutUrl(key, req.mimeType(),
        Duration.ofMinutes(15), req.size());

    return ResponseEntity.ok(new UploadUrlResponse(url, key));
}
```

---

## 2. 메시지 send 시 첨부 INSERT

```java
@Transactional
public Message sendWithAttachment(RoomId room, UserId user, AttachmentRequest req) {
    // magic byte 검증
    var head = s3.readBytes(req.key(), 0, 16);
    var detected = tika.detect(head);
    require(allowedMimes.contains(detected));

    var msg = Message.create(MessageId.next(), room, user, seq,
        req.type(), null, Map.of("attachmentKey", req.key()), null, null, now);
    messages.save(msg);

    var attachment = new Attachment(
        AttachmentId.next(), msg.id(), req.type(), req.url(),
        req.thumbnailUrl(), req.fileName(), req.size(), detected,
        req.width(), req.height(), req.durationMs(), null, "STANDARD", now);
    attachments.save(attachment);

    return msg;
}
```

---

## 3. Lambda thumbnail (S3 PUT event)

```javascript
// Lambda handler
exports.handler = async (event) => {
    for (const record of event.Records) {
        const key = record.s3.object.key;
        if (key.endsWith('-thumb')) continue;
        const obj = await s3.getObject(...);
        const thumb = await sharp(obj.Body).resize(200, 200).jpeg().toBuffer();
        await s3.putObject({ Bucket: ..., Key: key + '-thumb', Body: thumb });
    }
};
```

---

## 4. 함정

- mime 만 FE 신뢰 → magic byte 검증.
- size limit 검증 안 함.
- presigned URL 의 size condition 없음 → 큰 file 업로드.

---

## 관련

- [[implementation|↑ hub]]
- [[../design-decisions/attachment-strategy]]
- [[../security/attachment-scan]]
- [[../database/attachments-table]]
