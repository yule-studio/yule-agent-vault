---
title: "Template + i18n — 변수 치환 + locale fallback"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:37:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - template
  - i18n
---

# Template + i18n — 변수 치환 + locale fallback

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault

- DB `notification_templates` 테이블 — locale + type 별 row.
- 변수 치환 — `{{userName}}` / `{{amount}}` / `{{orderNumber}}`.
- locale fallback: `ko-KR → en-US (default)`.
- 채널 별 다른 template (push: title+body / email: subject+html).

---

## 2. Schema

```sql
CREATE TABLE notification_templates (
    id              CHAR(26) PRIMARY KEY,
    template_key    VARCHAR(100) NOT NULL,      -- "COMMENT_RECEIVED"
    channel_type    VARCHAR(20) NOT NULL,        -- "FCM" / "EMAIL" / ...
    locale          VARCHAR(10) NOT NULL,
    title           VARCHAR(200),                 -- push: title / email: subject
    body            TEXT NOT NULL,                -- push: body / email: html
    deeplink        VARCHAR(500),                 -- push: deeplink schema
    variables       TEXT[],                       -- 필수 변수 검증
    version         INTEGER NOT NULL DEFAULT 1,
    active          BOOLEAN NOT NULL DEFAULT TRUE,
    updated_by      CHAR(26),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (template_key, channel_type, locale, version)
);
```

---

## 3. 코드

```java
@Component
public class TemplateRenderer {

    private final NotificationTemplateRepository repo;
    private final Cache<TemplateLookup, NotificationTemplate> cache;

    public RenderedNotification render(NotificationOutboxRow row, UserDevice device) {
        var template = lookup(row.templateKey(), device.channelType(), device.locale());

        // {{userName}} → 실제 값
        var title = substitute(template.title(), row.variables());
        var body = substitute(template.body(), row.variables());

        return new RenderedNotification(title, body, template.deeplink());
    }

    private NotificationTemplate lookup(String key, ChannelType ch, String locale) {
        // 1. exact match (ko-KR)
        // 2. language fallback (ko)
        // 3. default (en-US)
        return repo.findByKeyAndChannelAndLocale(key, ch, locale)
            .or(() -> repo.findByKeyAndChannelAndLocale(key, ch, languageOnly(locale)))
            .or(() -> repo.findByKeyAndChannelAndLocale(key, ch, "en-US"))
            .orElseThrow();
    }

    private String substitute(String template, Map<String, Object> vars) {
        // Thymeleaf / Mustache / 단순 ${var}
        var result = template;
        for (var entry : vars.entrySet()) {
            result = result.replace("{{" + entry.getKey() + "}}",
                                     String.valueOf(entry.getValue()));
        }
        return result;
    }
}
```

---

## 4. 예시 template

```
key: COMMENT_RECEIVED
channel: FCM
locale: ko-KR
title: "{{commenterName}}님의 댓글"
body: "{{commenterName}}님이 회원님의 글에 댓글을 달았어요"
deeplink: "app://posts/{{postId}}"
variables: [commenterName, postId]

key: COMMENT_RECEIVED
channel: EMAIL
locale: ko-KR
title: "[YULE] {{commenterName}}님의 새 댓글"
body: "<html>...html template...</html>"
```

---

## 5. 함정

### 함정 1 — 변수 치환 검증 없음
template 의 `{{userName}}` 인데 row 의 variables 에 userName 없음 → runtime 에러.
→ template.variables 와 비교 검증.

### 함정 2 — XSS
사용자 입력 (`commenterName`) 이 HTML 안 → 스크립트 실행 위험.
→ HTML escape.

### 함정 3 — locale fallback 없음
잘못된 locale → 발송 실패.
→ ko → en fallback.

### 함정 4 — template 변경 audit 없음
누가 언제 변경했는지 X.
→ version + updated_by + audit row.

### 함정 5 — cache 무효화 없음
template 변경 후 옛 cache 사용.
→ Redis pub/sub 으로 invalidate.

---

## 6. 다른 컨텍스트

- Notion / Linear: Markdown template + 변수.
- Stripe: 자체 template editor + i18n 자동 (20+ language).

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[../database/notification-templates-table]]
- [[../implementation/template-impl]]
