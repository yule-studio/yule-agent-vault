---
title: "PCI-DSS — 카드번호 / CVC 서버 저장 X"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:57:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - security
  - pci-dss
---

# PCI-DSS — 카드번호 / CVC 서버 저장 X ★

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[security|↑ hub]]**

> PCI-DSS = Payment Card Industry Data Security Standard.
> 본 vault: **PG 가 책임** — 서버는 카드 정보 절대 X.

---

## 1. 본 vault 정책

| 데이터 | 서버 저장 | 처리 |
| --- | --- | --- |
| PAN (카드번호 16자리) | **절대 X** | PG SDK 만 처리 (iframe) |
| CVV (3자리) | **절대 X** | PG SDK 만 |
| 카드 만료일 | **절대 X** | PG SDK 만 |
| 카드 소유자 이름 | **절대 X** | PG SDK 만 |
| `pgPaymentKey` (PG 발급 토큰) | O | DB |
| 카드사 (BC / 신한 / KB) | O | 표시 용 |
| 마지막 4자리 (last4) | O (옵션) | "1234" 표시 |
| 결제 금액 / 시간 | O | 회계 |

---

## 2. 왜

- **PCI-DSS Level 1 심사**: 카드 저장 = 수억원 보안 심사 + 매년 갱신.
- 보안 사고 시 회사 종료 위험.
- → PG SDK 가 카드 정보 받음 → PG 가 PCI-DSS 처리.

## 3. 안 하면

| 잘못 | 결과 |
| --- | --- |
| 카드번호 DB 저장 | PCI-DSS Level 1 심사 (수억원) + 매년 갱신 |
| 로그에 카드번호 print | 로그 유출 시 사용자 카드 도용 |
| 카드번호 mask 안 함 (마지막 4자리만) | 카드사 / PG 협약 위반 |
| 우리 서버에서 카드 발급 (가상카드 등) | PCI-DSS + 금융업 라이센스 |

## 4. 안전 패턴 (PG SDK)

```
[FE iframe]      [PG 도메인]        [서버]
     │
     ├── PG SDK script src=PG 도메인
     │   (PG 의 iframe 으로 카드 입력창)
     │
     │       사용자 → PG iframe 에 카드 입력
     │       (PG 의 도메인 내에서만)
     │       PG 가 즉시 paymentKey 발급
     │
     ├── paymentKey ← SDK callback
     │
     └── /payments/confirm { paymentKey, orderId, amount }
                          ↓
                       서버 → PG /confirm API
                       서버 가 카드번호 본 적 없음 ✓
```

## 5. 함정

### 함정 1 — 카드번호 로그 print
DEBUG 로그 / 에러 log 에 출력 — 로그 유출 사고.
→ Logback 의 patternLayout 에서 PAN 패턴 mask filter.

### 함정 2 — 카드번호 form input
서버 controller 가 `String cardNumber` 받음.
→ FE controller method signature 에 카드 정보 없음.

### 함정 3 — last4 외에 다른 자리 저장
mask 했어도 6+4 (BIN + last4) 저장 = PCI-DSS 위반.
→ last4 만.

### 함정 4 — 토큰화 (자체)
자체 카드 token 만들 ↔ 카드 정보 매핑 보관 — 같은 위험.
→ PG 가 발급한 token 만.

---

## 6. 관련

- [[security|↑ hub]]
- [[../design-decisions/pg-selection]]
- [[../design-decisions/payment-flow]]
- [[idempotency-key]]
- [[webhook-signature]]
