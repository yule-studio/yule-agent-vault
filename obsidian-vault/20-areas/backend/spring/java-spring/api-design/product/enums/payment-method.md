---
title: "PaymentMethod enum"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:46:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - enum
---

# PaymentMethod enum

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[enums|↑ hub]]**

---

## 1. 값

```java
public enum PaymentMethod {
    CARD,              // 신용 / 체크 카드
    BANK_TRANSFER,     // 계좌이체 / 가상계좌
    MOBILE,            // 휴대폰 결제
    EASY_PAY;          // 토스페이 / 카카오페이 / 네이버페이 / 페이코
}
```

## 2. 메서드 별 정책

| Method | 정산 주기 | 환불 시간 | 부정 결제 위험 |
| --- | --- | --- | --- |
| CARD | D+3 | 1~3일 | 중간 (3DS 인증) |
| BANK_TRANSFER (가상계좌) | D+1 | 1일 (계좌 환급) | 낮음 |
| MOBILE | D+5~7 | 2~5일 (월 정산) | 낮음 |
| EASY_PAY | D+3 | 1~3일 | 낮음 (간편결제 인증) |

## 3. UX 우선순위 (한국)

```
1. EASY_PAY (토스/카카오/네이버) — 1탭
2. CARD — 입력 (브라우저 자동완성)
3. BANK_TRANSFER — 가상계좌 (offline)
4. MOBILE — 통신사 인증 (오프라인 대안)
```

## 4. 관련

- [[enums|↑ hub]]
- [[../domain-model/payment-aggregate]]
- [[../design-decisions/pg-selection]]
