---
title: "패스워드 해시 알고리즘 — Argon2id"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:10:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - password
  - hashing
---

# 패스워드 해시 알고리즘 — Argon2id

**[[design-decisions|↑ design-decisions hub]]**

> "비밀번호를 어떻게 저장하나" — 이 결정 하나가 **DB 백업 유출 시 모든 user 의 계정 운명** 을 결정.

---

## 1. 본 vault 결정

**Argon2id (m=64MB, t=3, p=4)**

- 라이브러리: `password4j` 1.8.x
- PHC 형식 저장: `$argon2id$v=19$m=65536,t=3,p=4$<salt>$<hash>`
- DB 컬럼: `password_hash VARCHAR(255)` (소셜은 NULL — [[../database/users-table#2.4]])

---

## 2. 비교 — 5개 알고리즘

| 알고리즘 | 권장 (2026) | GPU 저항 | 메모리 hard | 한국 SaaS 채택률 |
| --- | --- | --- | --- | --- |
| **Argon2id** | ✅ OWASP 1순위 | ✅ | ✅ | ↑ |
| **bcrypt** | ✅ 대안 | △ | ❌ | ✅ 가장 많음 (legacy) |
| **scrypt** | △ | ✅ | ✅ | 적음 |
| **PBKDF2-HMAC-SHA256** | △ | ❌ | ❌ | NIST 표준 (정부 / 금융) |
| SHA-256 / MD5 / SHA-1 단독 | ❌❌❌ | ❌ | ❌ | 절대 X |

---

## 3. 각 알고리즘의 "왜" — 4구조

### 3.1 Argon2id (본 vault 선택)

**왜 필요한가 (다른 알고리즘 대비 우위)**
- 2015 Password Hashing Competition 우승.
- GPU / ASIC 공격 저항: 메모리 hard (m=64MB) → GPU 의 병렬화 한계.
- argon2i (메모리 hard) + argon2d (GPU 저항) 의 hybrid = argon2id.
- OWASP 2024 권장 1순위.

**파라미터 — m=64MB, t=3, p=4 의 이유**
- m (메모리) = 65536 KB = 64MB. GPU 가 한 번에 수천 thread 돌릴 때 메모리 부담 ↑ → 병렬화 한계.
- t (iterations) = 3. CPU 시간 약 100-300ms 목표.
- p (parallelism) = 4. 일반 서버의 코어 수 (4 코어 기준).

**안 하면 무슨 문제**
- bcrypt 만 사용 시 → GPU 로 초당 수천 회 가능 (vs Argon2id 의 수십 회).
- 2024년 GPU 클러스터 = 약한 bcrypt 비밀번호 7시간 내 brute force 가능 (NVIDIA RTX 4090 기준).

**대안과 왜 안 됨**
- bcrypt — 메모리 hard 아님. GPU 공격에 상대적 약함.
- scrypt — Argon2id 보다 검증된 사례 적음.
- PBKDF2 — 메모리 hard 아님. 정부 표준이지만 보안성 ↓.

**트레이드오프**
- 검증 비용 = 100-300ms / 회 → 로그인 처리 시간 ↑.
- 메모리 64MB × 동시 로그인 수. 동시 100 로그인 = 6.4GB 메모리.
- 작은 서버 (Lambda, IoT) 에선 m=16MB / m=4MB 로 조정.

---

### 3.2 bcrypt

**왜 적합한 케이스**
- 가장 오래 검증된 (1999~) — 사례 풍부.
- Spring Security 기본 (`BCryptPasswordEncoder`).
- 한국 legacy 시스템 절대 다수가 이미 사용 중.

**왜 안 되는 케이스**
- 메모리 hard 아님 — GPU / ASIC 공격에 Argon2id 보다 약함.
- 72-byte 입력 제한 — 더 긴 비밀번호는 silent truncate (보안 사고 가능).

**대안과 왜 안 됨**
- bcrypt 의 cost=12 → ~200ms / 회. 적당히 안전하지만 Argon2id 가 같은 시간에 더 안전.

**트레이드오프**
- 신규 프로젝트 = Argon2id.
- 운영 중 시스템 = bcrypt 유지 + `needsRehash` 로 점진적 Argon2id 마이그레이션.

---

### 3.3 PBKDF2-HMAC-SHA256

**왜 적합한 케이스**
- NIST 표준 (SP 800-132) — 정부 / 금융 / FIPS 140-2 compliance.
- iterations ≥ 600,000 (OWASP 권장 2024).

**왜 안 되는 케이스**
- 메모리 hard 아님 — GPU 공격에 매우 취약.
- 한국 KISA / FIPS 호환 의무 없으면 사용 안 함.

**트레이드오프**
- compliance 우선 시 사용. 보안성은 Argon2id 보다 낮음.

---

### 3.4 SHA-256 / MD5 / SHA-1 (단독)

**왜 절대 안 됨**
- GPU 로 초당 수십억 회 시도 가능.
- Rainbow table — 1억 비밀번호의 hash 가 미리 계산된 DB 공유.
- 2026년 기준 = 8자리 영숫자 비밀번호는 1분 안에 brute force.

**언제 사용**
- 본 vault 의 **token_hash / phone_hash** — random / high-entropy 입력. brute force 불가능.
- 비밀번호 (low-entropy 입력) 에는 절대 X.

---

## 4. Argon2id 파라미터 결정 — 상세

### 4.1 서버 종류별 권장

| 사용처 | m (KB) | t | p | 검증 시간 |
| --- | --- | --- | --- | --- |
| 일반 서버 (4코어, 2026) | **65536 (64MB)** | **3** | **4** | 100-300ms |
| 작은 서버 / Lambda | 16384 (16MB) | 3 | 1 | 50-100ms |
| 모바일 / IoT | 4096 (4MB) | 3 | 1 | 20-50ms |
| 강한 보안 (금융) | 131072 (128MB) | 4 | 4 | 300-600ms |

### 4.2 파라미터 튜닝 원칙

**왜 검증 시간 100-300ms 목표**
- 너무 짧음 (< 50ms) — brute force 시간 ↓ → 보안성 ↓.
- 너무 김 (> 500ms) — 로그인 UX 부담 + DoS 위험 (단일 요청이 서버 자원 점유).
- 100-300ms = OWASP 권장 균형.

**왜 m 우선 증가**
- t / p 증가는 검증 시간 ↑ 만 시킴.
- m 증가 = GPU / ASIC 공격 비용 ↑↑ (병렬화 제약).
- 우선 순위: m > t > p.

### 4.3 운영 중 파라미터 강화

```java
// 옛 hash (m=32MB) 였을 때 — 새 m=64MB 로 강화
public boolean needsRehash(String hash) {
    var current = Argon2Function.getInstanceFromHash(hash);
    return current.getMemory() < 65536
        || current.getIterations() < 3
        || current.getParallelism() < 4;
}

// 로그인 성공 시 자동 rehash
public void login(...) {
    if (encoder.matches(rawPassword, user.passwordHash().value())) {
        if (encoder.needsRehash(user.passwordHash().value())) {
            user.changePassword(new PasswordHash(encoder.encode(rawPassword)));
            users.save(user);
        }
        // 로그인 성공
    }
}
```

**왜 점진적 (한 번에 모두 X)**
- 한 번에 모두 = 사용자가 비밀번호 알아야 (서버는 hash 만).
- 로그인 시 = 사용자가 평문 입력 → 새 hash 계산 → 점진적 업그레이드.

---

## 5. 라이브러리 — password4j 1.8

```kotlin
implementation("com.password4j:password4j:1.8.2")
```

**왜 password4j (Spring Security `BCryptPasswordEncoder` 가 아님)**
- Spring Security 의 `Argon2PasswordEncoder` 도 있지만 password4j 가 더 명시적 / 사용 편함.
- PHC 형식 (`$argon2id$v=19$m=...`) 자동 처리.
- 알고리즘 변경 / 파라미터 검증 (`needsRehash`) API 우수.

```java
@Component
public class Argon2PasswordEncoder implements PasswordEncoder {
    private final Argon2Function hasher = Argon2Function.getInstance(65536, 3, 4, 32, Argon2.ID);

    @Override
    public String encode(String plain) {
        return Password.hash(plain).addRandomSalt(16).with(hasher).getResult();
    }
    @Override
    public boolean matches(String plain, String hash) {
        return Password.check(plain, hash)
                       .with(Argon2Function.getInstanceFromHash(hash));
    }
}
```

자세히: [[../security#4 알고리즘 선정]] · [[../domain-model/password-hash-vo]].

---

## 6. 비밀번호 정책

```yaml
password-policy:
  min-length: 8
  max-length: 128
  required-classes:                   # 최소 3개
    - lowercase
    - uppercase
    - digit
    - special
  pwned-check: true                   # haveibeenpwned API
```

### 6.1 왜 min-length 8

- OWASP 2024 권장 최소.
- 짧으면 (4-6) — Argon2id 도 brute force 가능 (조합 = 100억 미만).

### 6.2 왜 max-length 128

- 너무 짧으면 (32) — passphrase ("correct horse battery staple") 사용자 제약.
- 너무 김 (1024+) — Argon2id 검증 시간 ↑ + DoS 위험.

### 6.3 왜 haveibeenpwned 체크

- 유출된 비밀번호 (5억+) DB 와 대조.
- "password123" 같은 흔한 비밀번호 reject.
- API: `https://api.pwnedpasswords.com/range/<sha1[0:5]>` — k-anonymity (앞 5자만 전송).

---

## 7. 함정 모음

### 함정 1 — bcrypt + 72 byte 초과 비밀번호
73 byte 부터 silent truncate. 사용자가 긴 비밀번호 입력해도 앞 72 만 검증.
→ Argon2id (제한 없음) 또는 bcrypt 앞에 SHA-256 전처리 (pepper pattern).

### 함정 2 — 단순 SHA-256 / MD5 사용
GPU 로 즉시 brute force.
→ 절대 X. KISA / 정보보호법 위반 + 사고 발생 시 책임.

### 함정 3 — Argon2id parameters 약함 (m=4MB)
GPU 공격 저항 ↓. 일반 서버 = m=64MB.

### 함정 4 — 검증 시간 너무 김 (1s+)
DoS 가능. 단일 IP 의 brute force 시도가 서버 마비.
→ 100-300ms + rate limit + lock.

### 함정 5 — `needsRehash` 안 함
파라미터 강화해도 옛 hash 그대로 → 효과 없음.
→ 로그인 성공 시 점진 rehash.

### 함정 6 — salt 직접 관리
실수로 같은 salt 사용 → rainbow table 가능.
→ 라이브러리가 random salt 자동 (PHC 형식에 임베드).

### 함정 7 — pepper 만 사용 (salt 없이)
DB 유출 + pepper 도난 시 모든 user 같은 평문 hash → rainbow table.
→ salt (per-user random) + pepper (server-wide secret, 옵션).

### 함정 8 — 비밀번호 변경 후 옛 RT revoke 안 함
공격자가 옛 RT 로 계속 access 갱신 → 변경 의미 없음.
→ password 변경 시 `refreshTokens.revokeAllForUser` 필수.

### 함정 9 — 응답 / 로그에 비밀번호 포함
실수로 log.info(password) → 로그 유출 시 전체 노출.
→ DTO 의 `toString()` 마스킹 + `@JsonIgnore`.

### 함정 10 — 로그인 실패 시 timing attack
"user 존재 X" 와 "비밀번호 틀림" 의 응답 시간 차이 → user enumeration.
→ user 없어도 dummy hash 검증 (같은 시간 소비).

### 함정 11 — bcrypt cost 너무 낮음 (cost=4)
brute force 너무 빠름.
→ cost ≥ 12 (2026).

### 함정 12 — pwned-check 안 함
"password123" 가입 통과 → 즉시 brute force 당함.
→ haveibeenpwned API 필수.

---

## 8. 다른 컨텍스트

### 8.1 금융 / 의료

```yaml
password-hash:
  algorithm: Argon2id
  params: { m: 131072, t: 4, p: 4 }   # 강화
  pepper: true                          # server-wide secret
  needsRehash: true                     # 매 로그인 점검
```

### 8.2 정부 / FIPS

```yaml
password-hash:
  algorithm: PBKDF2-HMAC-SHA256
  iterations: 600000
  reason: NIST SP 800-132 / FIPS 140-2 compliance
```

### 8.3 모바일 / IoT

```yaml
password-hash:
  algorithm: Argon2id
  params: { m: 4096, t: 3, p: 1 }       # 메모리 ↓
```

---

## 9. 관련

- [[design-decisions|↑ hub]]
- [[../domain-model/password-hash-vo]] — PasswordHash VO + 형식 검증
- [[../security#4]] — 알고리즘 선정
- [[../database/users-table#2.4]] — DB 컬럼 정책
- [[../login-impl]] · [[../password-reset-impl]]
- 외부 — OWASP Password Storage Cheat Sheet (2024)
