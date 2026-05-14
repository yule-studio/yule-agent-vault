---
title: "Template 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:02:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, template]
---

# Template 구현

**[[implementation|↑ hub]]**

---

## 1. TemplateRenderer

```java
@Component
@RequiredArgsConstructor
public class TemplateRenderer {

    private final NotificationTemplateRepository repo;
    private final Cache<TemplateLookup, NotificationTemplate> cache;
    private final HtmlSanitizer sanitizer;

    public RenderedNotification render(Notification n, UserDevice device) {
        var locale = device != null ? device.locale() : "ko-KR";
        var channel = device != null ? device.channelType() : ChannelType.IN_APP;

        var template = lookup(n.templateKey(), channel, locale);
        var title = substitute(template.title(), n.variables(), channel);
        var body = substitute(template.body(), n.variables(), channel);
        var deeplink = substitute(template.deeplink(), n.variables(), channel);

        return new RenderedNotification(title, body, deeplink);
    }

    private NotificationTemplate lookup(TemplateKey key, ChannelType ch, String locale) {
        // exact → language → default fallback
        return findExact(key, ch, locale)
            .or(() -> findExact(key, ch, languageOnly(locale)))
            .or(() -> findExact(key, ch, "en-US"))
            .orElseThrow(() -> new TemplateNotFoundException(key, ch, locale));
    }

    private String substitute(String template, Map<String, Object> vars, ChannelType ch) {
        if (template == null) return null;

        var result = template;
        for (var entry : vars.entrySet()) {
            var key = "{{" + entry.getKey() + "}}";
            var value = String.valueOf(entry.getValue());

            // HTML / push 채널 마다 escape 다르게
            if (ch == ChannelType.EMAIL) {
                value = HtmlEscape.escapeHtml4(value);
            }

            result = result.replace(key, value);
        }
        return result;
    }
}
```

---

## 2. Cache (TemplateLookup key)

```java
@Bean
public Cache<TemplateLookup, NotificationTemplate> templateCache() {
    return Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .build();
}

public record TemplateLookup(TemplateKey key, ChannelType channel, String locale) {}
```

→ template 변경 시 Redis pub/sub 으로 invalidate 가능.

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/template-i18n]]
- [[../database/notification-templates-table]]
