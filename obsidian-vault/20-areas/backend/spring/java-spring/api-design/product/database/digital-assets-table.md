---
title: "digital_assets 테이블 — 원본 디지털 자산 (책 PDF / 강의)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:57:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - digital
  - book
---

# digital_assets 테이블 — 원본 디지털 자산 (책 PDF / 강의)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> 원본 PDF / 영상 — admin 업로드. 사용자 별 마스킹 사본 ≠ 원본.

---

## 1. Schema

```sql
-- V36__create_digital_assets.sql
CREATE TABLE digital_assets (
    id              CHAR(26) PRIMARY KEY,
    product_id      CHAR(26) NOT NULL REFERENCES products(id),
    asset_type      VARCHAR(20) NOT NULL,        -- PDF / MP4 / EPUB / ZIP
    storage_provider VARCHAR(20) NOT NULL,       -- GDRIVE / S3
    storage_file_id VARCHAR(200) NOT NULL,        -- GDrive fileId / S3 key
    file_size       BIGINT NOT NULL,
    mime_type       VARCHAR(50) NOT NULL,
    checksum_sha256 CHAR(64) NOT NULL,            -- 무결성 검증
    title           VARCHAR(200) NOT NULL,
    metadata        JSONB,                        -- { "pages": 350, "isbn": "...", "author": "..." }
    watermark_enabled BOOLEAN NOT NULL DEFAULT TRUE,
    uploaded_by     CHAR(26) NOT NULL,
    uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at      TIMESTAMPTZ
);

CREATE INDEX ix_assets_product ON digital_assets (product_id) WHERE deleted_at IS NULL;
CREATE UNIQUE INDEX ux_assets_storage_provider_file
    ON digital_assets (storage_provider, storage_file_id);
```

---

## 2. 컬럼 "왜"

### 2.1 `storage_provider` + `storage_file_id`

- GDrive / S3 추상화 — 변경 가능.
- file_id = GDrive 의 webContentLink / S3 의 key.

### 2.2 `checksum_sha256`

- 업로드 무결성 검증.
- worker 가 워터마크 적용 전 원본 검증.

### 2.3 `watermark_enabled`

- 일부 자산 (샘플 / 무료) 은 워터마크 X.
- 책 / 강의는 항상 true.

### 2.4 `metadata JSONB`

- 책: pages, isbn, author, publisher.
- 강의: duration, resolution.
- 검색 / 표시 용.

### 2.5 `deleted_at` (soft)

- 옛 구매자의 마스킹 사본 (deliveries) 은 원본 link 필요.
- hard delete X.

---

## 3. 업로드 흐름

```
1. admin → POST /api/v1/admin/products/{id}/assets/upload-url
2. server → GDrive resumable upload URL (5분)
3. admin 직접 업로드 (대용량 100MB+)
4. admin → POST /api/v1/admin/products/{id}/assets
   { storage_file_id, file_size, checksum_sha256, title }
5. server → digital_assets INSERT
6. server async: checksum 재검증 (download + sha256)
```

자세히: [[../implementation/digital-delivery-impl#upload]].

---

## 4. 함정

### 함정 1 — storage_file_id 의 URL 직접 노출
사용자가 admin URL 알면 원본 다운.
→ private permission + service account only.

### 함정 2 — checksum X
업로드 변조 / 손상 감지 X.
→ sha256 + 재검증.

### 함정 3 — watermark_enabled hardcode
모든 자산에 적용 → 샘플 / 무료도 워터마크.
→ 컬럼 분리.

### 함정 4 — soft delete X
옛 구매자의 deliveries 원본 link 깨짐.
→ deleted_at.

---

## 5. 관련

- [[database|↑ hub]]
- [[digital-deliveries-table]]
- [[products-table]]
- [[../design-decisions/digital-delivery-policy]]
- [[../security/digital-watermarking]]
