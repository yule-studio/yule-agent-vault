---
title: "모더 권한 구현 — admin action + audit"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:45:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - security
  - moderation
---

# 모더 권한 구현 — admin action + audit

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[security|↑ security hub]]**

> Admin 의 모더 권한 + audit. 정책: [[../design-decisions/moderation-policy]].

---

## 1. 권한 매트릭스

| Action | USER | ADMIN | MASTER |
| --- | --- | --- | --- |
| 자기 글 hide / restore | ✅ | ✅ | ✅ |
| 다른 user 글 hide / restore | ❌ | ✅ | ✅ |
| 다른 user 글 delete | ❌ | ✅ | ✅ |
| Admin role 부여 | ❌ | ❌ | ✅ |
| audit log 조회 | ❌ | ✅ | ✅ |

---

## 2. 구현

```java
@RestController
@RequestMapping("/api/v1/admin")
@PreAuthorize("hasRole('ADMIN')")
@RequiredArgsConstructor
public class AdminModerationController {

    private final ModerationService service;

    @Operation(summary = "post 강제 hide")
    @PatchMapping("/posts/{postId}/status")
    public void moderatePost(
        @PathVariable PostId postId,
        @Valid @RequestBody ModerationRequest req,
        @AuthenticationPrincipal AuthUser admin
    ) {
        service.moderatePost(admin.id(), postId, req.action(), req.reason());
    }
}

@Service
@RequiredArgsConstructor
public class ModerationService {

    private final PostRepository posts;
    private final ModerationAuditRepository audit;
    private final NotificationOutbox outbox;

    @Transactional
    public void moderatePost(UserId adminId, PostId postId, ModerationAction action, String reason) {
        var post = posts.findById(postId).orElseThrow();
        var oldStatus = post.status();

        switch (action) {
            case HIDE -> post.hide(reason, Instant.now());
            case RESTORE -> post.restore(Instant.now());
            case DELETE -> post.delete(Instant.now());
        }
        posts.save(post);

        // audit 필수
        audit.save(new ModerationAuditLog(
            ids.next(),
            postId.value(), TargetType.POST,
            action.name(), oldStatus.name(), post.status().name(),
            adminId, reason, Instant.now()
        ));

        // 작성자 알림
        outbox.enqueue(NotificationFactory.moderationNotice(
            post.authorId(), postId, action, reason));
    }
}
```

---

## 3. Audit Log

```sql
CREATE TABLE moderation_audit_log (
    id           CHAR(26) PRIMARY KEY,
    target_id    CHAR(26) NOT NULL,
    target_type  VARCHAR(20) NOT NULL,                -- POST / COMMENT
    action       VARCHAR(30) NOT NULL,                -- AUTO_HIDDEN / ADMIN_HIDE / ADMIN_RESTORE / ADMIN_DELETE
    old_status   VARCHAR(20),
    new_status   VARCHAR(20),
    admin_id     CHAR(26),                            -- AUTO 시 NULL
    reason       TEXT,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_mod_audit_target ON moderation_audit_log (target_id, created_at DESC);
CREATE INDEX ix_mod_audit_admin ON moderation_audit_log (admin_id, created_at DESC);
```

### 3.1 왜 audit 필수

- admin 의 임의 처리 추적.
- 분쟁 시 입증.
- admin 자체 abuse 감지.

---

## 4. 자동 hide (admin X)

```java
@TransactionalEventListener(phase = AFTER_COMMIT)
public void onReportSubmitted(ReportSubmitted event) {
    long count = reports.countByTarget(event.targetId(), event.targetType());
    if (count >= 5) {
        autoHide(event.targetId(), event.targetType());
    }
}

@Transactional
public void autoHide(TargetId targetId, TargetType type) {
    if (type == POST) {
        var post = posts.findById(targetId.toPostId()).orElseThrow();
        if (post.status() != PostStatus.PUBLISHED) return;     // idempotent
        post.hide("5회 신고로 자동 hide", Instant.now());
        posts.save(post);
    } else {
        // comment
    }
    audit.save(new ModerationAuditLog(..., adminId=null /* SYSTEM */, ...));
    outbox.enqueue(NotificationFactory.autoHidden(...));
}
```

---

## 5. Admin Dashboard 권한

```java
@GetMapping("/admin/reports")
@PreAuthorize("hasRole('ADMIN')")
public Page<ReportResponse> listReports(
    @RequestParam ReportStatus status,
    @RequestParam(required = false) String cursor,
    @RequestParam(defaultValue = "20") int limit
) {
    // PENDING / AUTO_HIDDEN 우선
    return reportService.findByStatus(status, cursor, limit);
}
```

---

## 6. 함정

### 함정 1 — admin action audit 없음
admin 의 임의 처리 추적 X.
→ moderation_audit_log 필수.

### 함정 2 — admin 의 자체 글 모더 가능
admin 자기 글 hide 후 audit X.
→ 모든 action audit + admin_id 기록.

### 함정 3 — 자동 hide 의 idempotency 없음
race 시 두 번 hide 시도.
→ status 검증 후 변경.

### 함정 4 — admin 권한 escalation
일반 user 가 admin endpoint 호출 가능.
→ @PreAuthorize 필수.

### 함정 5 — Admin role 변경 audit 없음
누가 admin 부여했나 추적 X.
→ role_change_audit 별도.

### 함정 6 — 작성자 알림 누락
모더 후 작성자가 자기 글 안 보임 → 항의.
→ NotificationOutbox enqueue.

---

## 7. 관련

- [[security|↑ hub]]
- [[../design-decisions/moderation-policy]] — 정책
- [[../database/reports-table]]
- [[audit-logging]]
