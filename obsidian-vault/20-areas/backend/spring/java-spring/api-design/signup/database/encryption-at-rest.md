---
title: "암호화 정책 — at-rest / column-level / hash 인덱스"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - encryption
  - pii
---

# 암호화 정책 — at-rest / column-level / hash 인덱스

**[[database|↑ database hub]]**

> PII (개인정보) 보호 3계층. 어떤 데이터에 어떤 강도의 보호를 적용할지의 결정.
> 잘못 적용하면 **법적 책임 (PIPC 과징금), 검색 불가, key 관리 폭탄** 이 한꺼번에 터진다.

---

## 1. 왜 암호화 정책이 필요한가

### 1.1 위협 모델

| 위협 | 어떤 암호화가 막는가 |
| --- | --- |
| 디스크 / 백업 도난 (물리적) | at-rest encryption (RDS) |
| 백업 파일 유출 (운영자 실수) | at-rest + column-level |
| SQL injection / 코드 취약점 | column-level + hash (검색용) |
| 내부자 (DBA / 운영자) 의 무단 조회 | column-level + access control |
| Application 메모리 dump | TLS in-transit + memory protection (별도) |

### 1.2 왜 단계별 (한 번에 모두 X)

- 모든 컬럼 암호화 → 검색 불가 + 성능 폭망 + key 관리 폭탄.
- 데이터의 민감도 / 위협에 맞는 보호 강도 적용.

---

## 2. 3 계층 보호 정책

### 2.1 분류

| 강도 | 적용 대상 | 방식 |
| --- | --- | --- |
| **L1: at-rest only** | 모든 컬럼 | RDS encryption (KMS) |
| **L2: at-rest + hash 인덱스** | phone, email (검색 필요) | RDS + 별도 hash 컬럼 |
| **L3: at-rest + column 암호화** | 주민번호, 카드번호 | RDS + pgcrypto / KMS envelope |

### 2.2 본 vault 의 정책

| 데이터 | 분류 | 강도 |
| --- | --- | --- |
| user.id, user.created_at, status | 비-PII | L1 (at-rest) |
| user.email | PII (식별자) | L2 (hash 인덱스 없이도 검색 가능 — lower(email) UNIQUE) |
| user.phone | PII | L2 (phone_hash 검색용) |
| user.password_hash | 이미 hash | L1 |
| 결제 카드번호 | 최강 PII | L3 (pgcrypto + KMS, 본 vault 외 별도 PG) |
| 주민번호 | 최강 PII (필요 시) | L3 |

---

## 3. L1 — RDS at-rest encryption

### 3.1 무엇

- AWS RDS / Aurora 의 KMS 기반 encryption.
- 디스크 / 백업 / snapshot 모두 자동 암호화.
- application 코드 변경 X (transparent).

### 3.2 왜 기본 적용

**위협 모델**
- 디스크 도난 (datacenter 침입) — 매우 드물지만 발생 시 모든 데이터 노출.
- 백업 파일 / snapshot 의 무단 복사 — S3 ACL 잘못 설정 시.
- RDS 인스턴스 폐기 후 디스크 재활용.

**무엇을 못 막는가**
- SQL injection / app bug → DB 가 정상 query 로 응답 → 평문 노출.
- 내부자가 DB 에 직접 query → 평문 노출.

**비용**
- 거의 무료 (AWS RDS encryption 은 추가 비용 없음).
- 성능 영향 < 5% (TPS).

### 3.3 설정

```hcl
# Terraform 예시
resource "aws_db_instance" "main" {
  engine          = "postgres"
  storage_encrypted = true
  kms_key_id      = aws_kms_key.rds.arn
  ...
}
```

**왜 KMS Customer Managed Key (default AWS-managed key 아님)**
- Customer Managed = key rotation 정책 / IAM 제어 가능.
- 감사 (CloudTrail) — 누가 어떤 key 사용했는지 추적.

### 3.4 안 하면 무슨 문제

- 백업 S3 bucket 이 public 으로 잘못 노출 → 전체 DB 평문.
- 한국 PIPC 의 "안전성 확보 조치 의무" 위반 → 과징금.

---

## 4. L2 — hash 인덱스 (검색용)

### 4.1 무엇

- 평문 컬럼 + 별도 hash 컬럼 (SHA-256).
- application 은 hash 로 검색, 결과 row 의 평문 사용.
- 본 vault: `users.phone` + `users.phone_hash`.

### 4.2 왜 hash 인덱스가 필요한가

**문제 상황**
- 휴대폰 번호로 로그인 — `WHERE phone = ?` 가 인덱스 lookup 해야.
- phone 을 암호화 (L3) 하면 → 매 query 마다 decrypt → 인덱스 X → 풀스캔.
- phone 평문 + 인덱스 → DB 백업 유출 시 모든 phone 노출.

**해결 (L2)**
- phone 평문 컬럼 (RDS encryption 으로 보호) + phone_hash 컬럼.
- 검색: `WHERE phone_hash = sha256(?)` — hash 인덱스 lookup.
- 평문은 결과 row 에 있어 알림톡 발송에 사용.

**왜 단순 SHA-256 (HMAC 또는 argon2 아님)**
- phone 의 entropy 가 작음 (한국 휴대폰 = 약 5천만 경우의 수).
- HMAC + secret 사용 시 secret rotation 시 모든 hash 재계산.
- argon2 는 매 query μs → ms → auth 병목.
- SHA-256 + DB 백업 유출 대비는 충분 (rainbow table 가능하지만 한국 phone 5천만 = 작은 보호).

**더 강한 보호 옵션**
- HMAC-SHA256 + server-side secret → rainbow table 방어. secret rotation 정책 필요.
- 본 vault: 단순 SHA-256 (DB 백업 유출 대비) + RDS encryption.

### 4.3 application 코드

```java
public Optional<User> findByPhone(String phone) {
    String normalized = normalizePhone(phone);
    String hash = sha256(normalized);
    return userRepo.findByPhoneHash(hash);
}
```

**왜 정규화 먼저**
- `010-1234-5678` vs `01012345678` vs `+821012345678` 모두 같은 hash 보장.
- 정규화 안 하면 같은 사람의 hash 가 여러 개 → 검색 실패.

### 4.4 함정

- phone 평문 컬럼 직접 검색 (`WHERE phone = ?`) — 인덱스 없으면 풀스캔. application 정책으로 차단.
- hash 만 저장 + 평문 X → 알림톡 발송 시 평문 필요 → 발송 불가.

---

## 5. L3 — column-level 암호화

### 5.1 무엇

- 평문 X. DB 에 암호화된 값만.
- application 이 read 시 decrypt, write 시 encrypt.
- 본 vault 외 별도 영역 (결제 / 본인인증) 에 적용.

### 5.2 옵션

| 옵션 | 키 관리 | 사용 |
| --- | --- | --- |
| `pgcrypto` (PG extension) | DB 안에 key — 위험 | `pgp_sym_encrypt('plain', key)` |
| Application 단 AES-256-GCM | env / KMS | Java `Cipher` 직접 |
| KMS envelope encryption | AWS KMS | DEK (DB) + KEK (KMS) |
| Vault / HSM | 외부 서비스 | API 호출 |

### 5.3 본 vault 권장 — KMS envelope encryption

```
[Application]
  encrypt(plain):
    DEK = randomDataKey()         // generate
    encrypted_plain = AES-GCM(plain, DEK)
    encrypted_DEK = KMS.encrypt(DEK)
    DB.save(encrypted_plain, encrypted_DEK)

  decrypt(row):
    DEK = KMS.decrypt(row.encrypted_DEK)
    plain = AES-GCM(row.encrypted_plain, DEK)
    return plain
```

**왜 envelope (master key 만 KMS 아님)**
- KMS 호출 비용 — 매 read/write 마다 KMS 호출 시 latency + cost.
- DEK 는 row 별 / 일 별 / 도메인 별 cache 가능.
- KMS 호출은 DEK 생성 / decrypt 시만.

**왜 pgcrypto 안 됨**
- DB 안에 key 가 있으면 → SQL injection 시 DB 전체 노출.
- key rotation 어려움.

### 5.4 언제 L3 가 필수

- 결제 카드번호 (PCI-DSS).
- 주민번호 (한국 — 수집 자체가 법적 제한).
- 의료 / 금융 / 본인 인증 정보.

### 5.5 검색의 한계

- 암호화된 컬럼은 검색 불가 (인덱스 X).
- 필요시 별도 hash 컬럼 (L2 와 같은 패턴).
- 부분 검색 / range 검색 = 불가 (homomorphic encryption 같은 고급 기법 필요).

---

## 6. 비교 표

| 보호 강도 | 위협 | 비용 | 검색 | 본 vault 적용 |
| --- | --- | --- | --- | --- |
| L1 RDS encryption | 디스크 / 백업 유출 | 무료 | ✅ | 모든 컬럼 |
| L2 hash 인덱스 | DB 백업 / SQL injection (부분) | 약간 | hash equality only | phone |
| L3 column 암호화 | 내부자 / app 취약점 | 키 관리 부담 | X (또는 별도 hash) | (결제 영역 외부) |

---

## 7. Key 관리

### 7.1 RDS KMS 키

- Customer Managed Key — IAM 으로 접근 제어.
- automatic rotation (yearly) — KMS 가 자동.
- 삭제 시 — 7-30일 cooldown (실수 방지).

### 7.2 Application 단 key (L3)

| key 종류 | 어디 | rotation |
| --- | --- | --- |
| KEK (Key Encryption Key) | KMS | yearly |
| DEK (Data Encryption Key) | DB (encrypted by KEK) | per data / per day |
| TLS 인증서 | ACM | yearly |
| JWT signing key | env / secret manager | 6 months |

### 7.3 왜 rotation

- key 도난 시 영향 범위 시간 ↓.
- 일부 compliance (PCI-DSS) 의 요구사항.

### 7.4 안 하면 무슨 문제

- 영구 key 사용 → 어디서든 leak 시 영구 노출.
- 운영 중 key 분실 → 백업도 복구 불가 (envelope encryption 시 KEK 분실 = 모든 데이터 복구 불가).

---

## 8. 한국 개인정보보호법 / GDPR

### 8.1 한국 PIPC

| 데이터 | 의무 |
| --- | --- |
| 주민번호 | 수집 자체 제한 + 별도 동의 (의료/금융 외 X) |
| 휴대폰, 이메일 | 안전성 확보 조치 (암호화 / 접근 제어 / audit) |
| 비밀번호 | 일방향 암호화 (SHA-256 이상) |
| 카드번호 | 암호화 보관 (PCI-DSS 준수) |

**왜 본 vault 의 정책이 이걸 충족**
- phone: at-rest (RDS) + hash 인덱스 → "안전성 확보 조치" 충족.
- password: argon2id (단방향 hash) → "일방향 암호화" 충족.
- 카드번호: 본 vault 안 저장 (PG 만 보관).

### 8.2 GDPR

| 요구 | 본 vault 대응 |
| --- | --- |
| Right to erasure | soft delete + anonymize, 30일 후 완전 파기 |
| Data minimization | 필요 최소만 (주민번호 X) |
| Encryption | L1 + L2 + L3 (필요 시) |
| Breach notification | 72시간 내 (감사 audit log + 알람) |

---

## 9. 모니터링

| 메트릭 | 알람 |
| --- | --- |
| KMS API call rate | 평상 baseline 대비 spike = 의심 활동 |
| DB encryption status | `Storage-Encrypted` AWS 상태 변경 → 알람 |
| Failed decrypt count | brute force 의심 |

---

## 10. 함정 모음

### 함정 1 — phone 평문 + 인덱스
DB 백업 유출 = 전체 phone 노출. PIPC 과징금 사례 다수.
→ hash 인덱스.

### 함정 2 — 모든 컬럼 L3 암호화
검색 / 성능 망함. 데이터 분류 후 강도 조정.
→ L1 기본 + 민감 컬럼만 L2/L3.

### 함정 3 — pgcrypto + DB 안 key
SQL injection 시 key 도 노출. KMS / envelope 가 안전.

### 함정 4 — KMS 키 분실
envelope encryption 시 KEK 없으면 모든 데이터 복구 불가.
→ KMS deletion cooldown + audit.

### 함정 5 — Rotation 없음
영구 key → leak 시 영구 노출.

### 함정 6 — phone hash 가 단순 SHA-256
rainbow table 가능 (한국 휴대폰 5천만).
→ 강한 보호 필요 시 HMAC-SHA256 + server secret.

### 함정 7 — 정규화 안 하고 hash
`010-1234-5678` 과 `01012345678` 의 hash 가 다름 → 검색 실패.
→ 항상 정규화 후 hash.

### 함정 8 — 결제 카드번호 본 vault 저장
PCI-DSS 적용 + 책임 폭증.
→ PG (Stripe, 토스, 아임포트 등) 에 저장. token 만 받음.

### 함정 9 — 주민번호 수집
한국 PIPC — 의료/금융 외 거의 불법. 수집 자체 회피.
→ 본인 인증은 본인인증 서비스 (PASS / NICE) 에 위임.

### 함정 10 — Audit log 없음
누가 언제 어떤 데이터 조회했는지 X → GDPR breach 시 영향 범위 분석 불가.
→ DB / application 단 query audit.

### 함정 11 — TLS 미사용
in-transit 평문 노출. HTTPS / TLS 1.2+ 필수.

### 함정 12 — 메모리 dump
평문이 메모리에 잠시 있을 때 dump = 노출. JVM heap dump 정책 / 보호.

---

## 11. 관련

- [[database|↑ database hub]]
- [[users-table]] — phone_hash 적용
- [[../security]] — auth 보안 전반
- [[../design-decisions]] — 알고리즘 선정
- 외부 — 한국 개인정보보호법, GDPR, PCI-DSS, OWASP Cryptographic Storage Cheat Sheet
