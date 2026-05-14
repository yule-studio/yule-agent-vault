---
title: "부가세 / 영수증 정책 — 10% inclusive + 전자 영수증"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - design-decisions
  - tax
---

# 부가세 / 영수증 정책 — 10% inclusive + 전자 영수증

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[design-decisions|↑ design-decisions hub]]**

> 부가세 표시 방법 (inclusive vs exclusive) + 영수증 / 세금계산서 발급 전략.

---

## 1. 본 vault 결정

- **KRW inclusive** — 표시 가격에 부가세 10% 포함 (대고객 표준).
- 영수증 자동 (전자 영수증 PDF + 이메일).
- 세금계산서 (사업자 구매) → 후처리 발급 (CS).

---

## 2. 왜 / 안 하면 / 대안 / 트레이드오프

### 2.1 왜 inclusive

- 한국 표준 (백화점 / 마트 가격표 = inclusive).
- 사용자 혼동 없음 (장바구니 = 결제 = 동일 금액).

### 2.2 안 하면 (exclusive)

| 잘못 | 문제 |
| --- | --- |
| 11000원 표시 → 결제 시 12100원 | 사용자 화남 |
| 환불 시 세금 분리 안 함 | 정산 mismatch |
| 면세 / 영세율 구분 없음 | 세무 신고 오류 |

### 2.3 대안

| 모델 | 적용 |
| --- | --- |
| **inclusive** ★ | 한국 표준 |
| exclusive | 미국 (sales tax 주별 다름) |
| inclusive + exclusive 토글 | enterprise B2B |
| Stripe-Tax 자동 | 글로벌 |

### 2.4 트레이드오프

- inclusive = UX ↑ + B2B 영수증 시 분리 필요.
- exclusive = 세금 명확 + UX ↓.

---

## 3. 데이터 구조

```sql
order_items (
    id,
    product_id,
    quantity,
    unit_price_krw,         -- 세 포함 (inclusive)
    tax_rate,               -- 0.10 (10%)
    tax_amount_krw,         -- 분리 저장 (회계용)
    ...
)
```

→ 표시는 inclusive, 회계는 tax 분리 컬럼.

---

## 4. 영수증

| Type | 발급 | 형태 |
| --- | --- | --- |
| 개인 (전자영수증) | 자동, 결제 즉시 | PDF + 이메일 |
| 사업자 (세금계산서) | CS 요청 시 | 홈택스 연동 (별도 API) |
| 현금영수증 | 자동, 결제 시 | 국세청 자동 발급 (PG 의존) |

---

## 5. 함정

### 함정 1 — inclusive / exclusive 혼동
DB amount 가 inclusive 인데 코드는 exclusive 가정.
→ 컬럼 명에 `_inclusive` 명시 + Money VO.

### 함정 2 — 면세 상품 (도서)
책 = 면세 (부가세 X).
→ `taxRate` 0 + 면세 표시.

### 함정 3 — 환불 시 세금 분리 X
정산 / 세무 신고 오류.
→ refund row 도 tax 분리.

자세히: [[../enums/currency]] · [[../pitfalls/payment-pitfalls]].

---

## 6. 다른 컨텍스트

### 6.1 글로벌 (Stripe-Tax)
주 / 국가 별 자동 계산 (Sales Tax / VAT).

### 6.2 B2B SaaS
세금계산서 자동 + 후불 invoice.

---

## 7. 관련

- [[design-decisions|↑ hub]]
- [[pricing-strategy]]
- [[settlement-policy]]
- [[../enums/currency]]
