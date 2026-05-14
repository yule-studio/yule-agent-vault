---
title: "notification requirements — 35 AC"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T21:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - notification
  - requirements
---

# notification requirements — 35 AC

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[notification|↑ hub]]**

---

## 1. Outbox 발송 보장

| # | AC | 우선 |
| --- | --- | --- |
| O-1 | 도메인 트랜잭션 안에서 outbox row INSERT (AFTER_COMMIT 시 발송) | P0 |
| O-2 | DB rollback 시 outbox 도 rollback (event 손실 / 중복 X) | P0 |
| O-3 | Worker SKIP LOCKED 로 같은 row 동시 처리 X | P0 |
| O-4 | 일시 실패 (5xx / timeout) 시 exp backoff retry max 5회 | P0 |
| O-5 | 영구 실패 (invalid token / bad email) 시 즉시 DLQ | P0 |
| O-6 | DLQ 행 admin replay endpoint | P1 |
| O-7 | 30일 지난 SENT outbox 자동 cleanup | P1 |

## 2. 사용자 preference

| # | AC | 우선 |
| --- | --- | --- |
| P-1 | type 별 ON/OFF (LIKE_RECEIVED / COMMENT / CHAT / ...) | P0 |
| P-2 | 강제 ON 알림 (보안 alert / 결제 / 모더) — preference 무시 | P0 |
| P-3 | quiet hours (22:00-08:00) 시 push 보류 (다음 morning batch) | P1 |
| P-4 | per-channel 분리 (push ON / email OFF 가능) | P1 |
| P-5 | 사용자 GET / PATCH /me/notification-preferences | P0 |

## 3. Device 관리

| # | AC | 우선 |
| --- | --- | --- |
| D-1 | 사용자별 device 다중 등록 (폰 + 태블릿 + 웹) | P0 |
| D-2 | FCM `UNREGISTERED` / APNs `BadDeviceToken` 응답 시 자동 삭제 | P0 |
| D-3 | 미사용 device 90일 후 자동 청소 | P1 |
| D-4 | login 시 device token 등록 / 갱신 | P0 |
| D-5 | logout 시 해당 device token 비활성화 | P0 |

## 4. Dedup / Rate limit

| # | AC | 우선 |
| --- | --- | --- |
| R-1 | 같은 event_id 1h 안 중복 발송 X | P0 |
| R-2 | per-user rate limit 50/h (강제 ON 제외) | P0 |
| R-3 | per-user per-type rate limit 20/h | P1 |
| R-4 | burst detection (10 / 10s) → 자동 throttle + 집계 | P1 |

## 5. Template / i18n

| # | AC | 우선 |
| --- | --- | --- |
| T-1 | template key 별 (NotificationType + locale) 분리 | P1 |
| T-2 | 변수 치환 ({{userName}} / {{amount}}) | P0 |
| T-3 | locale fallback (ko-KR → en-US 기본) | P1 |
| T-4 | template 변경 audit (5년) | P2 |

## 6. 채널 별

| # | AC | 우선 |
| --- | --- | --- |
| C-1 | FCM: Android 다중 device 동시 발송 | P0 |
| C-2 | APNs: iOS HTTP/2 + JWT auth | P0 |
| C-3 | SES: From verify + DKIM + SPF + DMARC | P0 |
| C-4 | WebPush: VAPID + RFC 8030 | P1 |
| C-5 | Slack: admin only — Incoming Webhook | P1 |
| C-6 | In-App: DB row + 사용자 unread count | P0 |

## 7. 보안 / Audit

| # | AC | 우선 |
| --- | --- | --- |
| S-1 | device token 평문 저장 X (KMS 암호화) | P0 |
| S-2 | 발송 audit (5년 — 분쟁 evidence) | P0 |
| S-3 | abuse 검출 (1 user 1000+ / day → block) | P1 |

## 8. 비기능

| 항목 | 목표 |
| --- | --- |
| outbox INSERT 부하 | 1000 / s |
| worker 처리량 | 500 / s |
| FCM p95 latency | < 2s |
| in-app 발송 p95 | < 100ms |
| 가용성 | 99.9% (channel 의존 — 자체 99.95%) |

---

## 9. Phase 매핑

| Phase | AC |
| --- | --- |
| F1 | O-1 ~ O-3, C-6 (in-app) |
| F2 | C-1 (FCM) |
| F3 | C-2 (APNs) |
| F4 | C-3 (SES) + T-1 ~ T-3 |
| F5 | C-4 (WebPush) + C-5 (Slack) |
| F6 | P-1 ~ P-5 + D-1 ~ D-5 + R-1 ~ R-4 |
| F7 | O-4 ~ O-7 (DLQ + replay) + S-1 ~ S-3 |
| F8+ | Kafka |

---

## 10. 관련

- [[notification|↑ hub]]
- [[testing/test-scenarios]]
- [[implementation-order]]
