---
title: "Test scenarios — AC 매핑"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:08:00+09:00
tags: [backend, java-spring, api-design, notification, testing]
---

# Test scenarios — AC 매핑

**[[testing|↑ hub]]**

---

| AC | 테스트 | 도구 |
| --- | --- | --- |
| O-1 outbox INSERT | OutboxListenerIT.payment_inserts | TC |
| O-2 도메인 rollback | OutboxListenerIT.rollback_skips | TC |
| O-3 SKIP LOCKED | WorkerConcurrencyIT.multi_worker | TC + threads |
| O-4 exp backoff | RetryPolicyTest | Unit |
| O-5 permanent → DLQ | WorkerIT.invalid_token_dlq | TC + WireMock |
| O-6 admin replay | DlqReplayIT | TC |
| P-1 type 별 ON/OFF | PreferenceIT.disabled_skip | TC |
| P-2 강제 ON | PreferenceIT.security_enforced | TC |
| P-3 quiet hours | PreferenceIT.quiet_delays | TC + Clock fixed |
| D-1 multi device | FcmChannelIT.multi_device | TC + MockApns |
| D-2 UNREGISTERED → deactivate | FcmChannelIT.unregistered_cleanup | WireMock FCM |
| D-3 90일 cleanup | DeviceCleanupIT | TC + Clock |
| R-1 event_id UNIQUE | OutboxDedupIT | TC |
| R-2 rate limit | RateLimiterIT.50_per_hour | TC + Redis |
| C-1 FCM 발송 | FcmChannelIT.success | WireMock |
| C-2 APNs 발송 | ApnsChannelIT.success | MockApnsServer |
| C-3 SES 발송 | SesChannelIT.success | LocalStack |
| T-2 변수 치환 | TemplateRendererTest | Unit |
| T-3 locale fallback | TemplateRendererTest.fallback | Unit |

---

## 관련

- [[testing|↑ hub]]
- [[../requirements]]
