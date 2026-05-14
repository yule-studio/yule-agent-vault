---
title: "Channel mock tests — FCM/APNs/SES"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:14:00+09:00
tags: [backend, java-spring, api-design, notification, testing, mock]
---

# Channel mock tests — FCM/APNs/SES

**[[testing|↑ hub]]**

---

## 1. FCM — WireMock (Firebase REST API mock)

```java
stubFor(post(urlMatching("/v1/projects/.+/messages:send"))
    .willReturn(okJson("""
        { "name": "projects/yule/messages/abc123" }
        """)));
```

또는 Firebase Admin SDK mock:
```java
@MockBean FirebaseMessaging messaging;
when(messaging.send(any())).thenReturn("msg-id");
```

---

## 2. APNs — Pushy MockApnsServer

```java
var mockServer = new MockApnsServerBuilder()
    .setServerCredentials(...)
    .build();
mockServer.start(0).get();

// 사용 — production 와 동일 API
```

---

## 3. SES — LocalStack

```java
@Container
static LocalStackContainer localstack = new LocalStackContainer(
        DockerImageName.parse("localstack/localstack:3.0"))
    .withServices(SES);

@BeforeAll static void setup() {
    sesClient.verifyEmailIdentity(VerifyEmailIdentityRequest.builder()
        .emailAddress("noreply@example.com").build());
}
```

---

## 4. 관련

- [[testing|↑ hub]]
- [[integration-tests]]
