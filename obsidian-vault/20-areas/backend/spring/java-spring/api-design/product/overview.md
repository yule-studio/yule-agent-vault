---
title: "product overview — end-to-end 흐름"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - overview
---

# product overview — end-to-end 흐름

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[product|↑ hub]]**

> 상품 등록 → 노출 → 주문 → PG 결제 → 배송 (실물/디지털) → 환불 까지 한 페이지.

---

## 1. 큰 그림

```mermaid
flowchart TB
    subgraph admin["관리자"]
        A1[상품 등록<br/>PHYSICAL/DIGITAL/BOOK]
        A2[옵션/재고/이미지]
        A3[가격/할인]
    end

    subgraph catalog["카탈로그"]
        C1[상품 목록]
        C2[검색/필터/정렬]
        C3[상품 상세]
    end

    subgraph buyer["구매자"]
        B1[장바구니]
        B2[주문 작성<br/>배송지/쿠폰]
        B3[PG 결제창]
        B4[결제 완료]
    end

    subgraph fulfill["이행"]
        F1[실물 — 택배]
        F2[디지털 — 다운로드<br/>마스킹 PDF + GDrive]
    end

    subgraph back["백오피스"]
        K1[환불]
        K2[정산]
        K3[대사]
    end

    admin --> catalog
    catalog --> buyer
    buyer --> fulfill
    fulfill --> back
```

---

## 2. 결제 흐름 (시퀀스)

```mermaid
sequenceDiagram
    autonumber
    participant U as 사용자
    participant FE as 프론트
    participant API as 서버
    participant DB
    participant PG as PG (Toss)
    participant SDK as PG SDK

    U->>FE: 결제하기 클릭
    FE->>API: POST /orders
    API->>DB: order INSERT (status=PENDING)<br/>재고 차감
    API-->>FE: orderId / amount

    FE->>SDK: 결제창 open(orderId, amount)
    U->>SDK: 카드/페이 입력
    SDK->>PG: 인증
    PG-->>SDK: paymentKey
    SDK-->>FE: success callback (paymentKey)

    FE->>API: POST /payments/confirm<br/>(paymentKey, orderId, amount)
    Note over API: Idempotency-Key 검증
    API->>API: amount 검증 (변조 방지)
    API->>PG: POST /v1/payments/confirm
    PG-->>API: APPROVED
    API->>DB: payment.approve()<br/>order.confirmPayment()
    API-->>FE: 200 OK

    Note over PG,API: 비동기 보강
    PG->>API: POST /webhooks/toss<br/>(서명 HMAC)
    API->>API: 서명 검증
    API->>DB: payment idempotent (이미 APPROVED 면 200)
```

자세히: [[design-decisions/payment-flow]] · [[security/webhook-signature]].

---

## 3. 디지털 배송 (책 PDF) 흐름 ★

```mermaid
sequenceDiagram
    participant API
    participant L as Listener
    participant W as Watermark Worker
    participant GD as Google Drive
    participant U as 사용자

    Note over API: PaymentApproved AFTER_COMMIT
    API->>L: event
    L->>DB: digital_delivery INSERT (status=PENDING)
    L->>W: enqueue (orderId, userId, bookId)

    W->>GD: 원본 PDF read
    W->>W: 사용자별 unique 워터마크 적용<br/>(userId+orderId+timestamp)
    W->>GD: 마스킹된 PDF write (사용자별 폴더)
    W->>DB: digital_delivery (driveFileId, status=READY)
    W->>L: DigitalDeliveryReady event
    L->>U: email + push (다운로드 링크)

    U->>API: GET /downloads/{token}
    API->>API: 토큰 검증 + 다운로드 횟수 확인
    API->>GD: 단기 presigned URL 발급
    API-->>U: 302 redirect → GDrive presigned
```

자세히: [[design-decisions/digital-delivery-policy]] · [[implementation/digital-delivery-impl]] · [[security/digital-watermarking]].

---

## 4. 환불 흐름

```mermaid
sequenceDiagram
    participant CS as CS
    participant API
    participant PG
    participant DB

    CS->>API: POST /payments/{id}/cancel (amount, reason)
    API->>DB: payment.cancel(amount) → refund row
    API->>PG: POST /payments/cancel (paymentKey, cancelAmount)
    PG-->>API: PARTIAL_CANCELED / CANCELED
    API->>DB: status update
    Note over API: 디지털 상품: 다운로드 access revoke
    Note over API: 실물: 반품 송장 발급
    API->>DB: settlement row 조정
```

자세히: [[design-decisions/refund-policy]] · [[implementation/refund-impl]].

---

## 5. 데이터 큰 ERD

```mermaid
erDiagram
    USERS ||--o{ ORDERS : places
    USERS ||--o{ DIGITAL_DELIVERIES : owns
    PRODUCTS ||--o{ PRODUCT_OPTIONS : has
    PRODUCTS ||--o{ INVENTORY : stocks
    PRODUCTS ||--o{ DIGITAL_ASSETS : has
    ORDERS ||--|{ ORDER_ITEMS : contains
    ORDERS ||--|| PAYMENTS : has
    PAYMENTS ||--o{ PAYMENT_TRANSACTIONS : tries
    PAYMENTS ||--o{ REFUNDS : refunded
    PAYMENTS ||--o{ WEBHOOK_EVENTS : receives
    ORDER_ITEMS }o--|| PRODUCTS : refers
    DIGITAL_ASSETS ||--o{ DIGITAL_DELIVERIES : delivers
```

자세히: [[database/database]].

---

## 6. 상태 머신 (한 화면)

```mermaid
stateDiagram-v2
    [*] --> ORDER_PENDING
    ORDER_PENDING --> ORDER_PAID: payment approved
    ORDER_PENDING --> ORDER_CANCELED: timeout 30m
    ORDER_PAID --> ORDER_FULFILLED: 배송 완료 / 다운로드 가능
    ORDER_FULFILLED --> ORDER_REFUNDED: 환불

    state PAYMENT {
        [*] --> READY
        READY --> IN_PROGRESS: redirect
        IN_PROGRESS --> DONE: confirm 200
        IN_PROGRESS --> FAILED: confirm fail
        DONE --> PARTIAL_CANCELED: refund(part)
        DONE --> CANCELED: refund(all)
        PARTIAL_CANCELED --> CANCELED: refund(rest)
    }
```

자세히: [[enums/order-status]] · [[enums/payment-status]].

---

## 7. 관련

- [[product|↑ hub]]
- [[prerequisites]] — 시작 전 준비
- [[requirements]] — Acceptance Criteria
- [[architecture]] — Hexagonal 패키지
- [[transactions]] — race 분석
- [[implementation-order]] — F0~F9
