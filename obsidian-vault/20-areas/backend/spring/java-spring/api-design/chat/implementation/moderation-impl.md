---
title: "Moderation 구현 (신고 + block)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:02:00+09:00
tags: [backend, java-spring, api-design, chat, implementation, moderation]
---

# Moderation 구현 (신고 + block)

**[[implementation|↑ hub]]**

---

## 1. Block

```java
@PostMapping("/api/v1/users/{userId}/block")
public void block(@PathVariable String userId,
                  @RequestBody BlockRequest req,
                  @AuthenticationPrincipal AuthUser me) {
    blockService.block(me.userId(), UserId.of(userId), req.reason());
}

@DeleteMapping("/api/v1/users/{userId}/block")
public void unblock(@PathVariable String userId,
                    @AuthenticationPrincipal AuthUser me) {
    blockService.unblock(me.userId(), UserId.of(userId));
}
```

```java
@Transactional
public void block(UserId blocker, UserId blocked, String reason) {
    require(!blocker.equals(blocked), "self");
    blocks.save(new Block(blocker, blocked, reason, clock.now()));
    cache.invalidate("blocks:" + blocker.value());
    cache.invalidate("blocks:" + blocked.value());
}
```

---

## 2. 신고

```java
@PostMapping("/api/v1/reports")
public void report(@RequestBody ReportRequest req,
                   @AuthenticationPrincipal AuthUser me) {
    reportService.report(
        me.userId(),
        req.targetType(),     // MESSAGE / USER / ROOM
        req.targetId(),
        req.reason());
}
```

```java
@Transactional
public void report(UserId reporter, ReportTarget type, String targetId, String reason) {
    require(!hasDuplicate(reporter, type, targetId), "already reported");

    reports.save(new Report(...));

    // 5회 신고 → 자동 모더
    var count = reports.countDistinctReporters(type, targetId);
    if (count >= 5) {
        if (type == ReportTarget.MESSAGE) {
            messageService.autoHide(MessageId.of(targetId), "AUTO_5_REPORTS");
        } else if (type == ReportTarget.USER) {
            userService.flagForReview(UserId.of(targetId));
        }
        slack.alert("auto-moderation triggered", Map.of("target", targetId));
    }
}
```

---

## 3. 관련

- [[implementation|↑ hub]]
- [[../security/blocked-user-filter]]
- [[../design-decisions/moderation-blocking]]
