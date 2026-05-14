---
title: "product_images 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - database
  - image
---

# product_images 테이블

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[database|↑ hub]]**

> 상품 이미지 — S3 presigned 업로드 + CloudFront CDN.

---

## 1. Schema

```sql
-- V24__create_product_images.sql
CREATE TABLE product_images (
    id          CHAR(26) PRIMARY KEY,
    product_id  CHAR(26) NOT NULL REFERENCES products(id),
    sku_id      CHAR(26) REFERENCES product_skus(id),    -- 옵션 별 이미지 (옵션)
    url         VARCHAR(500) NOT NULL,                    -- S3 / CloudFront
    alt_text    VARCHAR(200),
    sort_order  INTEGER NOT NULL DEFAULT 0,
    is_primary  BOOLEAN NOT NULL DEFAULT FALSE,
    width       INTEGER,
    height      INTEGER,
    file_size   BIGINT,
    mime_type   VARCHAR(50),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX ix_images_product_sort ON product_images (product_id, sort_order);
CREATE UNIQUE INDEX ux_image_primary ON product_images (product_id) WHERE is_primary;
```

---

## 2. 컬럼 "왜"

### 2.1 `sku_id` (nullable)

- NULL = product 전체 대표 이미지.
- 값 = 옵션 (color 별 red 사진 / blue 사진).

### 2.2 `is_primary` + partial UNIQUE

- product 당 primary 1개.
- 카탈로그 목록에서 primary 우선 표시.

### 2.3 `width / height / file_size / mime_type`

- 카탈로그 페이지 layout 미리 알기 (CLS 방지).
- audit (사용자 업로드 시 검증 결과 보존).

---

## 3. 업로드 흐름

```
1. admin → POST /api/v1/products/{id}/images/upload-url
2. server → S3 presigned PUT URL (5분 TTL)
3. admin → S3 직접 PUT (대용량 file)
4. admin → POST /api/v1/products/{id}/images { url, sort_order }
5. server → product_images INSERT
```

자세히: [[../../file-upload-s3|↗ file-upload-s3]].

---

## 4. 함정

### 함정 1 — S3 직접 URL 노출
S3 url 변경 시 lock-in.
→ CloudFront URL 표준화.

### 함정 2 — 이미지 크기 검증 X
10GB 이미지 업로드.
→ presigned URL 의 condition (size max 10MB).

### 함정 3 — 옵션 별 이미지 없음
red / blue 사진 같음.
→ sku_id NULL 사용 (product 공용).

### 함정 4 — primary 2개
UNIQUE 없음 → race.
→ partial UNIQUE.

---

## 5. 관련

- [[database|↑ hub]]
- [[products-table]]
- [[../../file-upload-s3|↗ file-upload-s3]]
