---
title: "product prerequisites — 시작 전 준비"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - prerequisites
---

# product prerequisites — 시작 전 준비

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[product|↑ hub]]**

> 이 레시피를 그대로 따라가기 위한 코드 외 준비. PG 계약 / 사업자 등록 / 외부 서비스 가입 등.

---

## 1. 기술 전제 (코드)

| 항목 | 버전 / 도구 | 비고 |
| --- | --- | --- |
| Java | 21 LTS | virtual thread 사용 검토 |
| Spring Boot | 3.3.x | |
| PostgreSQL | 16 | JSONB / partial index 활용 |
| Redis | 7 | 재고 캐시 / 분산락 / idempotency |
| 분산 락 | Redisson | [[../distributed-lock]] |
| Auth | signup recipe | [[../signup/security/security]] |
| 이미지 / 파일 | S3 + CloudFront | [[../file-upload-s3]] |
| 결제 | PG SDK + REST | [[design-decisions/pg-selection]] |
| 디지털 책 | Google Drive API + PDFBox | [[implementation/digital-delivery-impl]] |
| 이메일 / push | SES + FCM/APNs | signup outbox 패턴 |

### 1.1 왜 PostgreSQL (MySQL 아님)

- 본 vault — JSONB (옵션 트리), partial index (status), `FOR UPDATE SKIP LOCKED` (재고 큐).
- MySQL: 인기 ↑ 지만 JSONB / partial index 미흡 → 우회 필요.

### 1.2 왜 Redis 필수

- 결제 idempotency key 의 TTL 캐시 (24h).
- 재고 차감의 atomic (Lua script).
- PG webhook 의 dedup 윈도우.

---

## 2. PG 계약 / 사업자 (비기술)

| 항목 | 한국 | 비고 |
| --- | --- | --- |
| 사업자등록증 | 필수 | 통신판매업 신고 추가 |
| PG 가맹 | Toss / KCP / 이니시스 / 카카오페이 / 네이버페이 중 선택 | [[design-decisions/pg-selection]] |
| 정산 계좌 | 사업자 명의 계좌 | KYC 검증 1주 |
| PCI-DSS | PG 가 보호 (서버는 카드번호 X) | [[security/pci-dss]] |
| 부가세 | 10% (간이/일반) | [[design-decisions/tax-strategy]] |
| 환불 정책 | 7일 이내 단순변심 (전자상거래법) | [[design-decisions/refund-policy]] |
| 청약철회 | 디지털: 다운로드 전만 환불 | [[design-decisions/digital-delivery-policy]] |
| 개인정보 | 결제 후 5년 보유 (전자상거래법) | [[security/pii-encryption]] |

### 2.1 왜 사업자 + 통신판매업

- 사업자 X → PG 가맹 거절.
- 통신판매업 신고 X → 공정위 과태료 + 결제 정지.

### 2.2 왜 PG 1곳 → 2곳 (이중화)

- 단일 PG 장애 시 매출 0.
- 본 vault: F5 에 1차 (Toss), F8 이후 2차 (KCP) 옵션.

자세히: [[design-decisions/pg-selection#11 이중화]].

---

## 3. 외부 서비스 가입

| 서비스 | 용도 | 비용 (월) | 비고 |
| --- | --- | --- | --- |
| **토스페이먼츠** | PG | 카드 2.8% + VAT | sandbox 무료 |
| **Google Workspace** | Drive API (책 마스킹) | $6/user | service account 권한 |
| **AWS S3 + CloudFront** | 이미지 / 일반 파일 | ~$20 | presigned URL |
| **AWS SES** | 영수증 / 알림 메일 | $0.10/1k | |
| **FCM / APNs** | 푸시 | 무료 | |
| **택배 API (CJ/한진/롯데)** | 송장 발급 | 건별 정산 | F7 |
| **사업자세관시스템 (선택)** | 해외 배송 | - | 글로벌 only |

### 3.1 왜 Google Drive (S3 아님) — 책 디지털 자산

- answer-be 패턴 (사용자가 알려준 영감).
- **왜 필요**: 책 PDF 는 100MB+ → 사용자별 마스킹 사본을 직접 보관해야 (S3 도 가능하지만 GDrive 는 viewer 기본 제공).
- **안 하면 문제**: S3 의 inline PDF viewer 별도 — UX ↓.
- **대안**: S3 + CloudFront + signed URL (월 cost ↓), 또는 Cloudfront 의 Lambda@Edge 워터마크.
- **트레이드오프**: GDrive 의 quota / rate limit (API 1000/100s). 사용자 폭증 시 S3 로 마이그.
- 자세히: [[design-decisions/digital-delivery-policy]].

---

## 4. 도메인 전제

| 결정 | 본 vault | 자세히 |
| --- | --- | --- |
| 다국적 | KRW only (1단계) → 다국적 (F후) | [[design-decisions/currency-strategy]] |
| 회원/비회원 | 회원 only (signup 필수) | |
| 단일 PG vs 멀티 | 단일 (Toss) — F5 / 멀티 — F8+ | [[design-decisions/pg-selection]] |
| 재고 차감 시점 | 주문 시점 | [[design-decisions/inventory-strategy]] |
| 결제 후 청약철회 | 디지털: 다운로드 전 only | [[design-decisions/refund-policy]] |
| 상품 type | PHYSICAL / DIGITAL / BOOK | [[enums/product-type]] |
| 정산 주기 | PG 입금 D+3 → 가맹점 D+5 | [[design-decisions/settlement-policy]] |

---

## 5. 보안 전제

| 항목 | 적용 | 자세히 |
| --- | --- | --- |
| 카드번호 / CVC | 서버 절대 저장 X | [[security/pci-dss]] |
| webhook 서명 | HMAC-SHA-256 | [[security/webhook-signature]] |
| 결제 amount 검증 | 서버에서 order.totalAmount 와 일치 | [[security/idempotency-key]] |
| PII 암호화 | 배송지 column-level (pgcrypto) | [[security/pii-encryption]] |
| 디지털 자산 보호 | 사용자별 워터마크 + 토큰 다운로드 | [[security/digital-watermarking]] |
| audit | admin action 5년 | [[security/audit-logging]] |

---

## 6. 인프라 / 배포

| 항목 | 본 vault | 자세히 |
| --- | --- | --- |
| DB 단일 / 분리 | 단일 → 결제 분리 (F후) | [[architecture]] |
| Migration | Flyway | |
| 트랜잭션 | 기본 READ_COMMITTED + 결제는 직접 락 | [[transactions]] |
| CI/CD | GitHub Actions | |
| 모니터링 | Prometheus + Grafana | [[operations/observability]] |
| 알람 | Slack #payment-alerts | [[operations/runbook]] |

---

## 7. 시작 전 체크리스트

- [ ] 사업자등록 + 통신판매업 신고 완료
- [ ] PG 1곳 선정 + sandbox key 발급 ([[design-decisions/pg-selection]])
- [ ] Google Drive service account + 폴더 권한
- [ ] signup recipe F0~F8 운영 중 (인증 통합)
- [ ] PostgreSQL 16 + Redis 7 인프라 셋업
- [ ] CloudFront + S3 (이미지)
- [ ] SES + FCM/APNs
- [ ] Slack alerts 채널
- [ ] 환불 정책 문서 (운영팀 + 법무 검토)
- [ ] PII 보유 기간 정책 (5년)

---

## 8. 관련

- [[product|↑ hub]]
- [[requirements]] — AC 목록
- [[architecture]] — Hexagonal 구조
- [[../signup/prerequisites|↗ signup prerequisites]] — 인증 전제
