---
title: "PII 암호화 — 배송지 column-level"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:07:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - security
  - pii
---

# PII 암호화 — 배송지 column-level

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[security|↑ hub]]**

---

## 1. 본 vault 정책

| 데이터 | 처리 |
| --- | --- |
| 배송지 (주소 / 이름 / 전화) | **column-level 암호화 (pgcrypto)** |
| 이메일 | 부분 마스킹 후 저장 (signup recipe 의 정책) |
| 카드 정보 | 서버 X (PCI-DSS — [[pci-dss]]) |
| 결제 금액 / 시간 | 평문 (회계) |
| 적립금 잔액 | 평문 |

---

## 2. 왜

- 배송지 = 개인 식별 가능 정보 (PII) — GDPR / 개인정보보호법.
- DB dump 유출 시 평문이면 직접 스토킹 가능.
- 암호화 = key 없으면 의미 X.

---

## 3. pgcrypto 통합

```sql
CREATE EXTENSION pgcrypto;

ALTER TABLE orders ADD COLUMN shipping_address_encrypted BYTEA;

-- 암호화 (write)
UPDATE orders SET shipping_address_encrypted =
    pgp_sym_encrypt(:json, :kmsKey)
WHERE id = :id;

-- 복호화 (read)
SELECT pgp_sym_decrypt(shipping_address_encrypted, :kmsKey) FROM orders WHERE id = ?;
```

→ key 는 AWS KMS / Vault, app 시작 시 fetch.

---

## 4. Java AttributeConverter

```java
@Converter
public class AddressEncryptedConverter implements AttributeConverter<Address, byte[]> {
    private final EncryptionService crypto;

    public byte[] convertToDatabaseColumn(Address a) {
        var json = mapper.writeValueAsString(a);
        return crypto.encrypt(json.getBytes(UTF_8));
    }

    public Address convertToEntityAttribute(byte[] b) {
        var json = new String(crypto.decrypt(b), UTF_8);
        return mapper.readValue(json, Address.class);
    }
}
```

---

## 5. 보유 기간 (전자상거래법)

| 데이터 | 기간 |
| --- | --- |
| 결제 / 주문 기록 | 5년 |
| 소비자 분쟁 | 3년 |
| 표시 / 광고 | 6개월 |

→ 기간 후 자동 cleanup cron (PII 컬럼 NULL).

---

## 6. 함정

### 함정 1 — JSON 컬럼 평문
JSONB 안에 평문 주소 — DB dump 유출 시 보임.
→ BYTEA + 암호화.

### 함정 2 — key DB / config 평문
key 노출 시 의미 X.
→ KMS / Vault.

### 함정 3 — 로그에 암호화 X
DEBUG log 에 주소 print.
→ Logback filter + masking.

### 함정 4 — 보유 기간 X
영구 누적 — 법적 위반.
→ cleanup cron.

### 함정 5 — encrypted 컬럼으로 검색
WHERE address LIKE '강남%' 안 됨 — 암호화 X.
→ 검색 X 정책 / blind index (별도 hash 컬럼).

---

## 7. 관련

- [[security|↑ hub]]
- [[../database/orders-table]]
- [[../domain-model/value-objects#address]]
- [[../../signup/security/pii-encryption|↗ signup PII]]
