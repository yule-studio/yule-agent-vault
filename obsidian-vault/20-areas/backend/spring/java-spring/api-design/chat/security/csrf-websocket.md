---
title: "CSRF (WebSocket) — Origin check + Same-Site cookie"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:26:00+09:00
tags: [backend, java-spring, api-design, chat, security, csrf]
---

# CSRF (WebSocket) — Origin check + Same-Site cookie

**[[security|↑ hub]]**

---

## 1. 본 vault — Origin 검증 + Same-Site cookie

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
            .setAllowedOrigins(
                "https://example.com",
                "https://app.example.com")
            .withSockJS();
    }
}
```

→ allowed origins 외 reject.

---

## 2. Cookie 정책

- JWT 는 Authorization header (cookie X) → CSRF 자체 영향 적음.
- cookie 사용 시:
  ```
  Set-Cookie: JSESSIONID=...; SameSite=Strict; Secure; HttpOnly
  ```

---

## 3. 함정

1. **`setAllowedOrigins("*")`** → 모든 site 가능 → CSRF.
2. **production 에 SockJS 만 fallback** — SockJS 의 HTTP polling 은 cookie 의존 → Same-Site 검토.
3. **CSRF token 안 씀 + cookie 인증** → attacker site 가 user 의 WS 호출.

---

## 관련

- [[security|↑ hub]]
- [[websocket-auth]]
