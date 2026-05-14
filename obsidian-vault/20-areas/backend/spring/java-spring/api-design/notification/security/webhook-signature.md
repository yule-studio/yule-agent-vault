---
title: "Channel response 검증 (FCM/APNs)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T22:36:00+09:00
tags: [backend, java-spring, api-design, notification, security]
---

# Channel response 검증 (FCM/APNs)

**[[security|↑ hub]]**

> notification 모듈은 webhook 수신 거의 X (FCM/APNs/SES 가 outbound).
> 수신 webhook: SES bounce notification + Slack interaction (admin response).

---

## 1. SES bounce / complaint webhook

```
AWS SES → SNS → 서버 webhook
```

- AWS SNS signature 검증 필요.
- bounce email → user_devices 의 email channel deactivate.

```java
@PostMapping("/api/v1/webhooks/ses-bounce")
public ResponseEntity<Void> sesBounce(@RequestBody String rawBody,
                                       @RequestHeader Map<String, String> headers) {
    // AWS SNS signature 검증
    if (!snsVerifier.verify(rawBody, headers)) return ResponseEntity.status(401).build();

    var event = parse(rawBody);
    if (event.notificationType().equals("Bounce")) {
        for (var recipient : event.recipients()) {
            deviceService.deactivateEmailChannel(recipient);
        }
    }
    return ResponseEntity.ok().build();
}
```

---

## 2. FCM / APNs 는 outbound only

- 응답이 동기 — 별도 webhook X.
- 단, FCM `UNREGISTERED` 응답을 webhook 형태로 보낼 수도 (옵션) — 일반적으로 동기 응답.

---

## 3. 관련

- [[security|↑ hub]]
- [[device-token-rotation]]
- [[../../product/security/webhook-signature|↗ product webhook]]
