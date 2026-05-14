---
title: "Runbook — 5 장애 시나리오"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:20:00+09:00
tags: [backend, java-spring, api-design, notification, operations, runbook]
---

# Runbook — 5 장애 시나리오

**[[operations|↑ hub]]**

---

## 1. FCM service account 만료 / 회수

```
탐지: FCM `THIRD_PARTY_AUTH_ERROR` spike
조치:
1. Firebase Console → Service Accounts
2. 새 private key 발급
3. Vault / Secrets Manager 갱신
4. app rolling restart
5. DLQ 의 FCM 실패 row replay
```

## 2. APNs .p8 key 만료 / 회수

```
탐지: 410 BadCertificate spike
조치:
1. Apple Developer → Keys → 새 key 생성
2. 옛 key 1주 유예 후 revoke
3. Vault 갱신 + 배포
4. DLQ replay
```

## 3. SES bounce rate 폭증

```
탐지: bounce rate > 5% / SNS 알림
원인: spam list / 잘못된 email DB
조치:
1. 의심 user group 식별 (recent registration / 같은 도메인)
2. bounce email 즉시 비활성화 (별도 cron)
3. AWS 에 reputation report 제출
4. 5% 이하 회복 후 알림 발송 재개
```

## 4. Outbox backlog (worker 정체)

```
탐지: outbox pending > 10000
조치:
1. worker scheduler 확인 (ShedLock 의 lock 가져갔는지)
2. worker concurrency 늘림
3. DLQ 누적 row admin replay
4. F8+ Kafka 도입 검토
```

## 5. DLQ 누적

```
탐지: DLQ count > 100
조치:
1. failure_reason 별 grouping
2. 패턴 (FCM_UNREGISTERED / SES_BOUNCE) → 자동 처리 가능
3. 분류 후 admin replay or resolve
4. resolved 90일 후 cleanup
```

---

## 6. on-call 절차

- 5분 안 ack
- 15분 안 상황 보고 #infra
- 1시간 안 외부 채널 (FCM / APNs / SES) 문의
- postmortem + 자동화

---

## 관련

- [[operations|↑ hub]]
- [[observability]]
- [[dlq-replay]]
