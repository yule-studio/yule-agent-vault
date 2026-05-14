---
title: "reports 테이블 — 신고"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T12:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - database
  - reports
---

# reports 테이블 — 신고

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[database|↑ database hub]]**

> 사용자 신고. 5회 자동 hide + admin review.

---

## 1. Schema

```sql
-- V12__create_reports.sql
CREATE TABLE reports (
    id           CHAR(26) PRIMARY KEY,
    reporter_id  CHAR(26) NOT NULL,
    target_id    CHAR(26) NOT NULL,                  -- post.id 또는 comment.id
    target_type  VARCHAR(20) NOT NULL,                -- POST / COMMENT
    reason       VARCHAR(30) NOT NULL,                -- enum
    detail       TEXT,                                 -- 추가 설명 (OTHER 시)
    status       VARCHAR(20) NOT NULL DEFAULT 'PENDING',
    admin_id     CHAR(26),                            -- review 한 admin
    admin_note   TEXT,
    resolved_at  TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (reporter_id, target_id, target_type),    -- 같은 사람 중복 신고 X

    CONSTRAINT chk_reports_target_type
        CHECK (target_type IN ('POST', 'COMMENT')),
    CONSTRAINT chk_reports_reason
        CHECK (reason IN ('SPAM', 'INAPPROPRIATE', 'HARASSMENT',
                          'HATE_SPEECH', 'VIOLENCE', 'COPYRIGHT',
                          'PERSONAL_INFO', 'OTHER')),
    CONSTRAINT chk_reports_status
        CHECK (status IN ('PENDING', 'AUTO_HIDDEN', 'DELETED', 'DISMISSED'))
);

CREATE INDEX ix_reports_target ON reports (target_id, target_type, status);
CREATE INDEX ix_reports_status_created ON reports (status, created_at DESC);
```

---

## 2. 컬럼 "왜"

### 2.1 UNIQUE (reporter_id, target_id, target_type)

- 같은 사람 중복 신고 차단.
- 5회 자동 hide threshold 의 fair count.

### 2.2 `target_type` + `target_id` (FK 없음)

- target 이 post 또는 comment — 단일 FK 불가.
- application 단 검증 (target 존재).

### 2.3 `reason` enum + CHECK

- 자유 text X — 통계 / 자동 처리 가능.
- `detail` 컬럼 = 추가 설명 (OTHER 선택 시).

### 2.4 status — 4-state

- PENDING — admin review 대기.
- AUTO_HIDDEN — 5회 신고 자동 hide.
- DELETED — admin 이 target 삭제.
- DISMISSED — admin 이 기각 (정상 글로 판단).

### 2.5 admin_id / admin_note

- 누가 / 언제 / 왜 처리했는지 audit.

---

## 3. 자동 hide 트리거

application 단:

```java
@Service
public class ReportService {

    @Transactional
    public void report(UserId reporterId, TargetId targetId, TargetType type, ReportReason reason) {
        try {
            reports.save(new Report(reporterId, targetId, type, reason));
        } catch (DataIntegrityViolationException e) {
            return;   // 중복 신고 silent
        }

        // 같은 target 의 신고 수 체크
        long count = reports.countByTarget(targetId, type);
        if (count >= 5) {
            autoHide(targetId, type);
        }
    }

    private void autoHide(TargetId targetId, TargetType type) {
        if (type == TargetType.POST) {
            posts.updateStatus(targetId.toPostId(), PostStatus.HIDDEN);
        } else {
            comments.updateStatus(targetId.toCommentId(), CommentStatus.HIDDEN);
        }
        notifier.notifyAuthor(targetId, "신고로 인한 임시 숨김");
    }
}
```

자세히: [[../design-decisions/moderation-policy#4 임계값]].

---

## 4. Admin Query

```sql
-- 처리 대기 신고 (오래된 순)
SELECT r.*, p.title, p.author_id
FROM reports r
LEFT JOIN posts p ON r.target_id = p.id AND r.target_type = 'POST'
WHERE r.status = 'PENDING'
ORDER BY r.created_at;

-- 같은 target 의 신고 수
SELECT target_id, target_type, COUNT(*) as report_count
FROM reports
WHERE status IN ('PENDING', 'AUTO_HIDDEN')
GROUP BY target_id, target_type
ORDER BY report_count DESC;
```

---

## 5. 함정

### 함정 1 — UNIQUE 누락
같은 user 가 5회 신고 → 자동 hide.
→ UNIQUE (reporter, target, type).

### 함정 2 — target FK
post 또는 comment — 단일 FK 불가. application 검증.

### 함정 3 — reason 자유 text
통계 / 자동 처리 X.
→ enum + CHECK.

### 함정 4 — admin_id 없음
누가 처리했는지 추적 X.
→ admin_id + admin_note + resolved_at.

### 함정 5 — DELETED 후 신고 row 삭제
audit 손실.
→ status=DISMISSED/DELETED 만 변경. row 보존.

### 함정 6 — 자동 hide 후 admin review 안 옴
HIDDEN 영구 — 정상 글 피해.
→ 24h SLA + 12h overdue alert.

### 함정 7 — 같은 IP 의 다중 계정 신고
신고 bombing.
→ 가중치 (정확도) 또는 IP / device 별 rate limit.

---

## 6. 관련

- [[database|↑ hub]]
- [[../design-decisions/moderation-policy]]
- [[../implementation/moderation-impl]] (todo)
- [[../security/audit-logging]] (todo)
