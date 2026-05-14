---
title: "XSS — message content sanitize"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:28:00+09:00
tags: [backend, java-spring, api-design, chat, security, xss]
---

# XSS — message content sanitize

**[[security|↑ hub]]**

---

## 1. 본 vault

- TEXT 메시지 = OWASP HTML Sanitizer 적용.
- 허용 tag: `b`, `i`, `u`, `s`, `code`, `pre`, `br`, `a` (href https only).
- markdown 일부 (옵션): bold / italic / code.

---

## 2. 코드

```java
@Component
public class ChatHtmlSanitizer {

    private final PolicyFactory policy = new HtmlPolicyBuilder()
        .allowElements("b", "i", "u", "s", "code", "pre", "br")
        .allowElements("a")
        .allowAttributes("href").onElements("a")
        .allowStandardUrlProtocols()
        .requireRelNofollowOnLinks()
        .toFactory();

    public String sanitize(String input) {
        if (input == null) return null;
        if (input.length() > 5000) throw new MessageTooLongException();
        return policy.sanitize(input);
    }
}
```

→ Message.create 에서 호출.

---

## 3. 함정

1. **sanitize 안 함** → `<script>` 메시지로 다른 user 에서 실행.
2. **FE 에서만 sanitize** → API 직접 호출 시 bypass.
3. **markdown render FE 에서** — DOMPurify 추가.
4. **mention link href** 검증 — `app://` schema 만.

---

## 관련

- [[security|↑ hub]]
- [[../domain-model/message-aggregate]]
- [[../../board/security/xss-defense|↗ board XSS]]
