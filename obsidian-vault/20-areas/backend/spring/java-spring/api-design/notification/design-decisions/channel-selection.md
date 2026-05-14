---
title: "채널 선택 — FCM / APNs / SES / WebPush / Slack / In-App"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:20:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - design-decisions
  - channel
---

# 채널 선택 — FCM / APNs / SES / WebPush / Slack / In-App ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ hub]]**

> 각 알림 type 에 어떤 채널 / 채널 조합 으로 발송할지 결정.

---

## 1. 본 vault 정책

| Notification Type | Primary | Fallback | 강제 ON |
| --- | --- | --- | --- |
| **PAYMENT_APPROVED** | FCM + APNs + In-App | Email | O |
| **PAYMENT_FAILED** | FCM + APNs + In-App | Email | O |
| **SECURITY_ALERT** (로그인 / 비밀번호 변경) | Email + FCM + APNs | In-App | **O (변경 X)** |
| **CHAT_MESSAGE** (오프라인) | FCM + APNs | WebPush | preference |
| **COMMENT_RECEIVED** | FCM + APNs + In-App | — | preference |
| **LIKE_RECEIVED** | FCM + APNs + In-App | — | preference (default OFF) |
| **MARKETING / PROMOTION** | Email | In-App | preference (default OFF) |
| **MODERATION_HIDDEN** (admin → user) | Email + In-App | — | **O** |
| **REFUND_PROCESSED** | Email + FCM + APNs + In-App | — | preference |
| **ADMIN_ALERT (fraud / 5xx)** | Slack | — | (admin 채널) |

---

## 2. 왜 / 안 하면 / 대안 / 트레이드오프

### 2.1 왜 type 별 채널 결정

**왜 필요**
- 결제 = 즉시 알림 필수 → push + email (오프라인 사용자도 봐야).
- 좋아요 = 사용자 OFF 가능 (spam 위험).
- admin 알림 = 사용자 인터페이스 X → Slack.
- → 일률적 채널 = UX / 비용 / 보안 균형 X.

**안 하면 어떤 문제**
- 모든 알림 email → 사용자 inbox 폭주 → spam 분류 → 중요 알림 누락.
- 모든 알림 push → app 못 깔린 사용자 의 보안 알림 누락.
- admin 알림 email → 운영팀 noise.

### 2.2 왜 다중 채널 (FCM + APNs + Email)

- **FCM only**: Android 사용자 의 iPhone 친구는 알림 X.
- **APNs only**: 반대.
- **FCM + APNs**: 양쪽 platform 모두 + 같은 사용자 의 다중 디바이스.
- **+ Email**: 오프라인 (장시간 미접속) 시 fallback.

### 2.3 왜 In-App 항상 포함 (가능한 경우)

- 사용자 가 push 무시 / OFF 해도 app 열면 봄.
- 알림 이력 (history) — push 는 즉시 disappear.
- 본 vault: `notifications` 테이블 → 사용자 화면 목록.

### 2.4 왜 강제 ON 알림

- 보안 alert / 결제 / 모더 = 사용자 권리 (transparency / 책임).
- 사용자 가 OFF → 회사 책임 (사기 / 분쟁).
- → preference 무시.

**안 하면 어떤 문제**
- 사용자 가 "결제 알림 OFF" 후 도용 결제 못 알아챔 → 분쟁.
- 모더 후 글 hidden — 사용자 항의.

### 2.5 대안 (전체 채널 전략)

| 모델 | 적용 |
| --- | --- |
| **type 별 다중 채널** ★ (본 vault) | 일반 SaaS |
| email-only | B2B / 단순 SaaS |
| push-only | 모바일 first |
| user-preference 만 (default 다중) | UX 강 |
| AI 학습 (사용자 행동 분석 후 채널 결정) | 큰 platform |

### 2.6 트레이드오프

- 다중 채널 = 발송 비용 ↑ (FCM 무료 but SES 유료).
- single 채널 = 사용자 누락 가능.
- 본 vault: **type 별 결정 + 사용자 preference override 가능** (강제 ON 제외).

---

## 3. ChannelRouter 의사결정 코드

```java
@Component
public class ChannelDecider {

    public List<ChannelType> decide(NotificationType type, UserId user) {
        var prefs = preferences.findByUser(user);
        var enforced = type.enforced();   // SECURITY / PAYMENT / MODERATION

        // 1. type 의 default 채널
        var defaultChannels = type.defaultChannels();

        // 2. 강제 ON 이면 preference 무시
        if (enforced) return defaultChannels;

        // 3. preference 확인 — 사용자 ON 한 채널 만
        return defaultChannels.stream()
            .filter(c -> prefs.isChannelEnabled(type, c))
            .toList();
    }
}
```

---

## 4. 함정

### 함정 1 — 모든 type 같은 채널
spam / 누락 / 비용 균형 X.
→ type 별 매트릭스.

### 함정 2 — 강제 ON 무시
사용자 가 결제 / 보안 알림 OFF.
→ enforced flag.

### 함정 3 — Email 위주 (push X)
모바일 first 사용자 무력.
→ push + email 둘 다.

### 함정 4 — admin 알림 사용자에게
운영 noise 가 사용자 화면.
→ Slack only.

### 함정 5 — In-App 빠짐
push 무시 후 이력 X.
→ In-App 항상 포함.

---

## 5. 다른 컨텍스트

### 5.1 카카오톡 / 라인
- chat = 자체 push (FCM/APNs) + 카톡 내부 channel
- 다른 알림 (예: 결제) = 자체 banner + push

### 5.2 Slack / Discord
- in-app (실시간) + push (모바일 fallback)

### 5.3 B2B SaaS (Notion / Linear)
- email 우선 + Slack 통합 (작업장 알림)
- push 는 모바일 only

### 5.4 게임
- push 우선 (이벤트 / 친구 요청)
- email 거의 X

---

## 6. 관련

- [[design-decisions|↑ hub]]
- [[user-preferences]]
- [[../enums/notification-type]]
- [[../enums/channel-type]]
- [[../implementation/channel-router-impl]]
