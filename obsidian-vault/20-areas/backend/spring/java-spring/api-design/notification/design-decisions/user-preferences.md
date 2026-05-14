---
title: "사용자 preference — type 별 ON/OFF + 강제 ON"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:28:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - preference
---

# 사용자 preference — type 별 ON/OFF + 강제 ON

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ hub]]**

---

## 1. 본 vault

- type × channel 의 2 차원 매트릭스 (예: COMMENT × PUSH = ON, COMMENT × EMAIL = OFF).
- 강제 ON type (변경 X): SECURITY / PAYMENT / MODERATION.
- quiet hours (22:00-08:00 사용자 timezone) — push 보류 + 다음 morning batch.

---

## 2. 왜 / 안 하면 / 대안

### 2.1 왜 type × channel 2 차원

**왜 필요**
- 사용자 마다 alert mix 다름 — "push 는 받지만 email 은 X".
- type 만 ON/OFF → email spam 분리 X.

**안 하면**
- 모든 type 단일 ON/OFF → 사용자가 spam 거부하면 모든 알림 OFF.

### 2.2 왜 강제 ON

- 보안 / 결제 / 모더 = transparency 의무 + 회사 책임.
- 사용자 OFF 시 사기 / 분쟁 책임.

### 2.3 왜 quiet hours

- 새벽 spam → 사용자 짜증 → app 삭제.
- push 보류 → morning batch (default 08:00 사용자 timezone).

**대안**
| 모델 | 적용 |
| --- | --- |
| **2D matrix + 강제 ON + quiet hours** ★ | 본 vault |
| type 만 (1D) | 단순 SaaS |
| channel 만 (1D) | 단순 SaaS |
| AI 학습 (행동 기반) | 큰 platform |

---

## 3. Schema

```sql
CREATE TABLE user_notification_preferences (
    user_id           CHAR(26) PRIMARY KEY,
    settings          JSONB NOT NULL DEFAULT '{}',     -- { "COMMENT": { "FCM": true, "EMAIL": false } }
    quiet_hours_start TIME,                             -- 22:00
    quiet_hours_end   TIME,                             -- 08:00
    quiet_timezone    VARCHAR(50) NOT NULL DEFAULT 'Asia/Seoul',
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.1 왜 JSONB

- type × channel 동적 (새 type 추가 시 schema 변경 X).
- 단점: query 복잡 — but settings.path 로 검색 가능.

---

## 4. API

```http
GET /api/v1/me/notification-preferences

{
  "COMMENT_RECEIVED":  { "FCM": true,  "EMAIL": false, "IN_APP": true },
  "LIKE_RECEIVED":     { "FCM": false, "EMAIL": false, "IN_APP": true },
  "PAYMENT_APPROVED":  { "enforced": true },
  "MARKETING":         { "FCM": false, "EMAIL": false, "IN_APP": false }
}

PATCH /api/v1/me/notification-preferences
{ "COMMENT_RECEIVED": { "FCM": false } }
```

---

## 5. 검증 로직

```java
public boolean shouldSend(NotificationType type, ChannelType ch, UserId user) {
    if (type.enforced()) return true;
    var prefs = repo.findByUser(user);
    return prefs.isEnabled(type, ch);
}

public boolean isInQuietHours(UserId user, Instant now) {
    var prefs = repo.findByUser(user);
    var localTime = now.atZone(ZoneId.of(prefs.timezone())).toLocalTime();
    return inRange(localTime, prefs.quietStart(), prefs.quietEnd());
}
```

→ quiet hours 내 → outbox 의 `next_attempt_at` 을 morning 으로 설정 (보류).

---

## 6. 함정

### 함정 1 — 강제 ON 없음
사용자 OFF 한 결제 알림 → 사기 분쟁.
→ enforced flag.

### 함정 2 — quiet hours 글로벌 timezone
사용자 미국 거주 — 한국 quiet hours 적용.
→ 사용자 timezone.

### 함정 3 — JSONB query 매번
요청 마다 DB hit.
→ Redis cache 5분.

### 함정 4 — preference 기본값 (모두 ON)
신규 사용자가 spam — 1주 후 app 삭제.
→ LIKE / MARKETING 은 default OFF.

---

## 7. 다른 컨텍스트

- 카카오톡: 친구 별 알림 ON/OFF (room 단위).
- Discord: server / channel 단위.
- Gmail: tab / label 별 필터.

---

## 8. 관련

- [[design-decisions|↑ hub]]
- [[channel-selection]]
- [[../database/user-preferences-table]]
- [[../implementation/preference-impl]]
