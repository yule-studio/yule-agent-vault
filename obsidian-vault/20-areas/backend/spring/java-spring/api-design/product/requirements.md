---
title: "product requirements — Acceptance Criteria"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - requirements
---

# product requirements — Acceptance Criteria

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[product|↑ hub]]**

> 9개 영역, 총 52 AC. 각 AC = 통합 테스트 1 케이스로 대응.

---

## 1. 상품 등록 (admin)

| # | AC | 우선 |
| --- | --- | --- |
| P-1 | admin 은 상품 (PHYSICAL/DIGITAL/BOOK) 등록 가능 | P0 |
| P-2 | 상품 상태: DRAFT → ACTIVE → SOLD_OUT → DISCONTINUED | P0 |
| P-3 | 상품 옵션 (size, color) — 옵션별 SKU + 재고 분리 | P1 |
| P-4 | 이미지 5장 + 동영상 1개, S3 presigned 업로드 | P0 |
| P-5 | XSS sanitization (상품 설명 HTML) | P0 |
| P-6 | 가격 변경 시 audit log + 이전 가격 보존 | P1 |
| P-7 | DISCONTINUED 상품은 카탈로그 / 검색 노출 X (직접 URL 만 가능) | P0 |

자세히: [[implementation/product-crud-impl]] · [[security/audit-logging]].

---

## 2. 카탈로그 / 검색

| # | AC | 우선 |
| --- | --- | --- |
| C-1 | 카테고리별 / 태그별 필터 | P0 |
| C-2 | 정렬: 최신 / 인기 / 가격↑↓ / 평점 | P0 |
| C-3 | 검색 (Phase 1: ILIKE → Phase 2: PostgreSQL FTS → Phase 3: ES) | P0 |
| C-4 | cursor-based 페이징 (OFFSET 금지) | P0 |
| C-5 | 가격대 필터 (range) | P1 |
| C-6 | 옵션 (size, color) 필터 | P1 |

자세히: [[../board/design-decisions/pagination-strategy|↗ cursor pagination]] · [[../product-search]] (흡수 예정).

---

## 3. 장바구니

| # | AC | 우선 |
| --- | --- | --- |
| K-1 | 회원만 (비회원 X 정책 — F0 단순화) | P0 |
| K-2 | 옵션별 SKU 단위 add/update/remove | P0 |
| K-3 | 최대 50 항목 / 사용자 | P1 |
| K-4 | 상품 가격 변경 시 장바구니에 알림 | P1 |
| K-5 | SOLD_OUT 상품은 장바구니 유지 (이력) + 결제 X | P0 |

자세히: [[../cart]] (흡수 예정).

---

## 4. 주문 / 재고

| # | AC | 우선 |
| --- | --- | --- |
| O-1 | 주문 생성 시 재고 차감 (atomic, Redis Lua) | P0 |
| O-2 | 재고 부족 시 409 + 부분 주문 X | P0 |
| O-3 | 동시성 100 요청 100 차감 (재고만큼만 성공) | P0 |
| O-4 | 결제 실패 / timeout 30m 시 재고 복원 | P0 |
| O-5 | 쿠폰 1개 + 적립금 적용 | P1 |
| O-6 | 배송지 입력 + 저장 (PII 암호화) | P0 |
| O-7 | 주문 상태: PENDING → PAID → FULFILLED → REFUNDED | P0 |

자세히: [[design-decisions/inventory-strategy]] · [[transactions]] · [[../order-stock]] (흡수 예정).

---

## 5. 결제 (PG)

| # | AC | 우선 |
| --- | --- | --- |
| PAY-1 | PG redirect → /confirm 서버 호출 → DB 트랜잭션 | P0 |
| PAY-2 | `Idempotency-Key` 헤더 — 같은 키 재호출 시 동일 응답 | P0 |
| PAY-3 | amount 검증 (클라가 보낸 amount 와 order.totalAmount 일치) | P0 |
| PAY-4 | webhook 서명 검증 (HMAC-SHA-256) | P0 |
| PAY-5 | webhook 중복 — payment_key dedup window 24h | P0 |
| PAY-6 | 결제 상태: READY → IN_PROGRESS → DONE / FAILED / CANCELED / PARTIAL_CANCELED | P0 |
| PAY-7 | PG 응답 시간 5s timeout → FAILED + 재고 복원 | P0 |
| PAY-8 | 카드번호 / CVC 서버 저장 X (PCI-DSS) | P0 |
| PAY-9 | 영수증 발행 (PDF + 이메일) | P1 |
| PAY-10 | 결제 시도 audit (payment_transactions table) | P1 |

자세히: [[design-decisions/payment-flow]] · [[security/webhook-signature]] · [[implementation/payment-confirm-impl]].

---

## 6. 환불

| # | AC | 우선 |
| --- | --- | --- |
| R-1 | DONE → 전체 환불 = CANCELED | P0 |
| R-2 | DONE → 부분 환불 = PARTIAL_CANCELED + 나머지 환불 가능 | P0 |
| R-3 | 환불 시 PG cancel API 호출 + 멱등 | P0 |
| R-4 | 디지털 상품: 다운로드 전만 환불 (다운로드 후 환불 X 정책) | P0 |
| R-5 | 실물 상품: 배송 전 100% / 배송 후 회수 후 환불 | P1 |
| R-6 | 환불 시 적립금 차감 (회수) | P1 |
| R-7 | 환불 audit (5년 보존) | P0 |

자세히: [[design-decisions/refund-policy]] · [[implementation/refund-impl]].

---

## 7. 디지털 배송 (책) ★

| # | AC | 우선 |
| --- | --- | --- |
| D-1 | 결제 승인 → digital_delivery row INSERT (status=PENDING) | P0 |
| D-2 | Worker 가 GDrive 원본 → 사용자별 unique 워터마크 PDF 생성 | P0 |
| D-3 | 워터마크 내용: userId + orderId + 구매일 + email 일부 | P0 |
| D-4 | 사용자별 GDrive 폴더 자동 생성 (folder per user) | P0 |
| D-5 | 다운로드 토큰 발급 (만료 7일, 사용 횟수 5회) | P0 |
| D-6 | 다운로드 시 audit (IP / UA / time) | P0 |
| D-7 | 환불 시 다운로드 access revoke (token 무효화 + GDrive permission 제거) | P0 |
| D-8 | 워터마크 worker 실패 시 retry (max 5회) + DEAD_LETTER | P0 |
| D-9 | 사용자 화면 — "내 라이브러리" (구매한 책 목록) | P1 |

자세히: [[design-decisions/digital-delivery-policy]] · [[implementation/digital-delivery-impl]] · [[security/digital-watermarking]].

---

## 8. 실물 배송

| # | AC | 우선 |
| --- | --- | --- |
| SH-1 | 결제 승인 → 배송 row INSERT (status=READY) | P0 |
| SH-2 | 송장 발급 (CJ/한진/롯데 API) → tracking_no 저장 | P0 |
| SH-3 | 배송 상태 webhook (PICKED_UP / IN_TRANSIT / DELIVERED) | P1 |
| SH-4 | 배송 완료 → order FULFILLED | P0 |
| SH-5 | 사용자 알림 (배송 시작 / 완료) | P1 |

자세히: [[implementation/physical-delivery-impl]].

---

## 9. 정산 / 운영

| # | AC | 우선 |
| --- | --- | --- |
| S-1 | PG webhook → payment 승인 정산 row | P0 |
| S-2 | 일/월 정산 batch (PG 입금 D+3 → 가맹점 D+5) | P1 |
| S-3 | 대사 (서버 payment 합계 = PG 정산 금액) — daily | P0 |
| S-4 | 환불 시 정산 차감 | P0 |
| S-5 | 메트릭: 매출 / 거래 / 환불율 / PG fee | P1 |

자세히: [[design-decisions/settlement-policy]] · [[operations/reconciliation]].

---

## 10. 우선순위 / Phase 매핑

| Phase | AC |
| --- | --- |
| F1 | P-1 ~ P-7 |
| F2 | C-1 ~ C-6 |
| F3 | K-1 ~ K-5 |
| F4 | O-1 ~ O-7 |
| F5 | PAY-1 ~ PAY-10 |
| F6 | D-1 ~ D-9 |
| F7 | SH-1 ~ SH-5 |
| F8 | R-1 ~ R-7 |
| F9 | S-1 ~ S-5 |

---

## 11. 비기능

| 항목 | 목표 |
| --- | --- |
| 상품 목록 p95 | < 200ms (캐시 hit 시 < 50ms) |
| 결제 confirm p95 | < 1.5s (PG 의존) |
| 디지털 워터마크 p95 | < 30s (100MB PDF) |
| 다운로드 p95 | < 500ms (GDrive presigned redirect) |
| availability | 99.9% (PG 의존 — 자체 99.95%) |
| 동시 결제 | 1000 TPS |

---

## 12. 관련

- [[product|↑ hub]]
- [[overview]]
- [[architecture]]
- [[implementation-order]]
- [[testing/test-scenarios]] — AC → 테스트 매핑
