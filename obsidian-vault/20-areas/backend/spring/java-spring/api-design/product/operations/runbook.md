---
title: "Runbook — 5가지 장애 시나리오"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T20:06:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - operations
  - runbook
---

# Runbook — 5가지 장애 시나리오

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[operations|↑ hub]]**

---

## 1. PG 장애 (Toss down)

```
탐지: 결제 confirm 5xx > 5% / Slack #payment-alerts P0
조치:
1. PG status page (status.tosspayments.com)
2. feature flag `pg.fallback.enabled` ON → KCP 사용 (F8+ 이중화 가정)
3. 사용자 안내 (banner)
4. 30분 후 대사 확인 + 복원
```

## 2. Kafka broker down

```
탐지: outbox pending > 1000 / Kafka lag > 10000
조치:
1. broker 노드 health check
2. broker 재시작 (k8s rolling)
3. outbox worker 가 자동 catch-up
4. DLQ 누적 확인 → admin replay
```

## 3. 디지털 워터마크 worker 장애

```
탐지: digital_deliveries 의 PENDING > 100 / 30분 이상
조치:
1. worker 로그 확인 (PDFBox / GDrive API rate)
2. GDrive quota 초과? → 일시 PAUSE + 다음 주기 대기
3. retry exp backoff 작동 확인
4. DEAD_LETTER 누적 시 admin manual 처리
```

## 4. 재고 mismatch (Redis vs DB)

```
탐지: 매 15분 cron → Slack alert
조치:
1. mismatch sku 식별
2. DB 의 on_hand 권위로 Redis 업데이트
3. 원인 (Redis 장애 / 차감 race) 분석
4. 필요 시 inventory_audit 추가
```

## 5. 환불 후 디지털 access 안 닫힘

```
탐지: 사용자 신고 (사용자가 환불 받고도 다운로드 가능)
조치:
1. delivery row 의 revoked_at 확인
2. GDrive permission 확인
3. token_hash NULL 검증
4. 누락이면 manual revoke + AFTER_COMMIT listener bug fix
```

---

## 6. on-call 절차

| 시간 | 대응 |
| --- | --- |
| 5분 안 | 알람 ack + 조치 시작 |
| 15분 안 | status / 진행 상황 #payment-alerts |
| 1시간 안 | 외부 (PG / 택배사) 문의 |
| 사후 | postmortem + 자동화 |

---

## 7. 관련

- [[operations|↑ hub]]
- [[observability]]
- [[../pitfalls/payment-pitfalls]]
- [[../pitfalls/digital-delivery-pitfalls]]
