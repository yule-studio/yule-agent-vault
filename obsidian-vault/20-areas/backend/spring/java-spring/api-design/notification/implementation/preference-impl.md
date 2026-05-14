---
title: "Preference 구현"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:00:00+09:00
tags: [backend, java-spring, api-design, notification, implementation, preference]
---

# Preference 구현

**[[implementation|↑ hub]]**

---

## 1. Controller

```java
@RestController
@RequestMapping("/api/v1/me/notification-preferences")
@RequiredArgsConstructor
public class PreferenceController {

    private final PreferenceService service;

    @GetMapping
    public ResponseEntity<PreferenceResponse> get(@AuthenticationPrincipal AuthUser auth) {
        return ResponseEntity.ok(service.get(auth.userId()));
    }

    @PatchMapping
    public ResponseEntity<Void> patch(@RequestBody Map<String, Map<String, Boolean>> patch,
                                       @AuthenticationPrincipal AuthUser auth) {
        service.patch(auth.userId(), patch);
        return ResponseEntity.ok().build();
    }
}
```

---

## 2. Service

```java
@Service
@RequiredArgsConstructor
public class PreferenceService {

    private final UserNotificationPreferenceRepository repo;
    private final Cache<UserId, UserNotificationPreference> cache;

    public boolean isEnabled(UserId user, NotificationType type, ChannelType ch) {
        if (type.enforced()) return true;
        var prefs = cache.get(user, k -> repo.findByUserId(k)
            .orElseGet(() -> repo.save(UserNotificationPreference.defaults(k))));
        return prefs.isEnabled(type, ch);
    }

    public boolean isInQuietHours(UserId user, Instant now) {
        var prefs = cache.get(user, ...);
        if (prefs.quietStart() == null) return false;
        var local = now.atZone(ZoneId.of(prefs.timezone())).toLocalTime();
        return inRange(local, prefs.quietStart(), prefs.quietEnd());
    }

    @Transactional
    public void patch(UserId user, Map<String, Map<String, Boolean>> patch) {
        var prefs = repo.findByUserId(user).orElseGet(() ->
            UserNotificationPreference.defaults(user));
        patch.forEach(prefs::set);
        repo.save(prefs);
        cache.invalidate(user);
    }
}
```

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../design-decisions/user-preferences]]
- [[../database/user-preferences-table]]
