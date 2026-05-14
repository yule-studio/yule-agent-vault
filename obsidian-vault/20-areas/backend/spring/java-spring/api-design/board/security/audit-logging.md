---
title: "Audit Logging — board 의 action 추적"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - security
  - audit
---

# Audit Logging — board 의 action 추적

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[security|↑ security hub]]**

> board 의 admin / user action 추적. signup audit + board 특화.

---

## 1. 무엇을 기록

| Event | 보유 기간 |
| --- | --- |
| POST_CREATED / POST_DELETED | 1년 |
| ADMIN_MODERATE (hide / restore / delete) | 5년 |
| AUTO_HIDDEN (5회 신고) | 5년 |
| REPORT_RESOLVED (admin 판단) | 5년 |
| USER_BLOCKED / UNBLOCKED | 1년 |
| ATTACHMENT_UPLOADED (size 큰 file) | 6개월 |

---

## 2. signup audit 의 board 확장

기존 `auth_audit_log` 테이블 ([[../../signup/security/audit-logging]]) 에 board action 추가:

```sql
-- 본 vault: 별도 board_audit_log 테이블 (각 도메인 분리)
CREATE TABLE board_audit_log (
    id            CHAR(26) PRIMARY KEY,
    event_type    VARCHAR(50) NOT NULL,
    actor_id      CHAR(26),                          -- user_id (AUTO 시 NULL)
    actor_role    VARCHAR(20),                        -- USER / ADMIN / SYSTEM
    target_id     CHAR(26),                          -- post / comment id
    target_type   VARCHAR(20),
    ip_address    VARCHAR(45),
    user_agent    VARCHAR(500),
    metadata      JSONB,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    INDEX (event_type, created_at DESC),
    INDEX (actor_id, created_at DESC),
    INDEX (target_id, target_type, created_at DESC)
);
```

→ signup audit 과 다른 테이블 — domain 분리.

---

## 3. Logging 구현

```java
@Component
@RequiredArgsConstructor
public class BoardAuditLogger {

    private final BoardAuditRepository repo;
    private final HttpServletRequest request;

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onPostCreated(PostCreated event) {
        repo.save(new BoardAuditLog(
            ids.next(), "POST_CREATED",
            event.authorId().value(), "USER",
            event.postId().value(), "POST",
            extractIp(request), request.getHeader("User-Agent"),
            Map.of("boardId", event.boardId().value()),
            Instant.now()
        ));
    }

    @TransactionalEventListener(phase = AFTER_COMMIT)
    public void onAdminAction(AdminModerationAction event) {
        repo.save(new BoardAuditLog(
            ids.next(), "ADMIN_" + event.action().name(),
            event.adminId().value(), "ADMIN",
            event.targetId().value(), event.targetType().name(),
            extractIp(request), request.getHeader("User-Agent"),
            Map.of("reason", event.reason(),
                   "oldStatus", event.oldStatus().name(),
                   "newStatus", event.newStatus().name()),
            Instant.now()
        ));
    }
}
```

---

## 4. 알람 / 메트릭

| 메트릭 | 알람 |
| --- | --- |
| ADMIN_DELETE 발생 | Slack #admin-actions |
| AUTO_HIDDEN spike (10분 5+) | Slack #security |
| 같은 admin 의 모더 폭주 (1h 50+) | Slack #security (admin abuse 의심) |

---

## 5. Cleanup

```sql
-- 1년 지난 POST_CREATED / POST_DELETED
DELETE FROM board_audit_log
WHERE event_type IN ('POST_CREATED', 'POST_DELETED')
  AND created_at < now() - INTERVAL '1 year';

-- 5년 지난 ADMIN action
DELETE FROM board_audit_log
WHERE event_type LIKE 'ADMIN_%'
  AND created_at < now() - INTERVAL '5 years';
```

자세히: [[../../signup/security/audit-logging]].

---

## 6. 함정

### 함정 1 — admin action audit 없음
abuse 추적 X.
→ 모든 admin action audit.

### 함정 2 — SYSTEM action audit X
AUTO_HIDDEN 의 시점 / 사유 모름.
→ actor_id=NULL + actor_role='SYSTEM'.

### 함정 3 — audit 가 비즈니스 트랜잭션 안
rollback 시 audit 도 사라짐.
→ AFTER_COMMIT 또는 REQUIRES_NEW.

### 함정 4 — PII 직접 저장
사용자 글 본문 audit 에.
→ 핵심 metadata 만 (ID).

### 함정 5 — 보유 기간 X
DB 무한 누적.
→ event_type 별 cleanup.

---

## 7. 관련

- [[security|↑ hub]]
- [[../../signup/security/audit-logging]] — auth audit
- [[moderation-impl]] — admin action
- [[../design-decisions/moderation-policy]]
