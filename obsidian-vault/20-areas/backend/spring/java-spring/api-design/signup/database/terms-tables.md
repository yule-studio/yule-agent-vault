---
title: "terms + user_terms_consent_history 테이블"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T03:04:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - database
  - terms
  - consent
---

# terms + user_terms_consent_history 테이블

**[[database|↑ database hub]]**

> 약관 버전 관리 + 사용자 동의 history. 한국 개인정보보호법 / GDPR 의 핵심 audit.
> 이 테이블이 부실하면 **법적 분쟁 시 입증 불가, 약관 변경 시 마이그레이션 폭탄, CS 응대 자료 부실** 이 한꺼번에 터진다.

---

## 1. 왜 별도 테이블 — `users.marketing_agreed BOOLEAN` 안 되나

```sql
-- ❌ 안티
ALTER TABLE users ADD COLUMN marketing_agreed BOOLEAN;
```

**왜 안 되는가 — 구체적 시나리오**

1. **약관 버전 변경 처리 X**
   - 2024년에 v1.0 동의 → 2025년 v2.0 배포 → 옛 동의자가 v2.0 동의했다는 보장 없음.
   - boolean 만으로는 "어떤 버전에 동의했는지" 모름.

2. **동의 시점 audit X**
   - 법적 분쟁 시 "이 사용자가 정확히 언제 동의했나?" — 분 단위로 입증 필요.
   - boolean 만으론 답할 수 없음 (`agreed_at` 별도 컬럼 추가? → 결국 history 테이블).

3. **철회 history X**
   - "동의했다가 철회했다가 다시 동의" 흐름 — 단일 boolean 으로 불가.
   - 마케팅 수신 동의는 자주 변경됨. 변경 history 가 audit / 분쟁 필수.

4. **여러 약관 = 컬럼 폭증**
   - 서비스 / 개인정보 / 마케팅 / 위치 / 광고 / 만 14세 등 — 5~10개.
   - `users` 테이블에 컬럼 10개 추가 = schema 비대 + ALTER TABLE 자주.

5. **약관 폐기 후 정보 손실**
   - v1.0 폐기 시 → 옛 동의자의 동의 사실까지 사라짐. audit 망함.

**결론**: 반드시 별도 테이블 + 버전 관리.

---

## 2. Schema

### 2.1 `terms` — 약관 자체 (버전별 row)

```sql
-- V2__create_terms.sql
CREATE TABLE terms (
    id           CHAR(26) PRIMARY KEY,
    code         VARCHAR(50)  NOT NULL,             -- 'service-terms' / 'privacy-policy' / 'marketing'
    version      VARCHAR(20)  NOT NULL,             -- 'v1.0', 'v1.1', 'v2.0'
    title        VARCHAR(200) NOT NULL,
    content      TEXT NOT NULL,
    required     BOOLEAN NOT NULL DEFAULT false,    -- 필수 동의 약관
    effective_at TIMESTAMPTZ NOT NULL,              -- 시행일
    use_yn       VARCHAR(1) NOT NULL DEFAULT 'Y',   -- 'Y' = 활성, 'N' = 폐기
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE UNIQUE INDEX ux_terms_code_version ON terms (code, version);
CREATE INDEX ix_terms_active_code ON terms (code, effective_at DESC) WHERE use_yn = 'Y';
```

### 2.2 `user_terms_consent_history` — 동의 history

```sql
-- V2__create_user_terms_consent_history.sql
CREATE TABLE user_terms_consent_history (
    id          CHAR(26) PRIMARY KEY,
    user_id     CHAR(26) NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    terms_id    CHAR(26) NOT NULL REFERENCES terms(id),
    consent_yn  VARCHAR(1) NOT NULL,                -- 'Y' / 'N'
    agreed_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address  VARCHAR(45),                         -- IPv6 max 45 char
    user_agent  VARCHAR(255)
);

CREATE INDEX ix_user_terms_consent_user ON user_terms_consent_history (user_id);
CREATE INDEX ix_user_terms_consent_terms ON user_terms_consent_history (terms_id);
CREATE INDEX ix_user_terms_consent_user_agreed
    ON user_terms_consent_history (user_id, agreed_at DESC);
```

---

## 3. 컬럼의 "왜" — 4구조

### 3.1 terms.`code VARCHAR(50)` — stable identifier

**왜 필요한가**
- `'service-terms'` 같은 의미적 id. 버전이 바뀌어도 같은 약관 = 같은 code.
- 검색: "현재 활성 서비스 약관" → `WHERE code='service-terms' AND use_yn='Y' ORDER BY effective_at DESC LIMIT 1`.

**안 하면 무슨 문제**
- code 없이 id 만 쓰면 → application 이 "현재 활성 service-terms 의 id 가 뭐지?" 매번 찾기. 매핑 비용.
- 새 버전 배포 시 application 의 hardcoded id 변경 필요.

**대안과 왜 안 됨**
- `type ENUM` (PostgreSQL native) → 약관 종류 추가 시 ALTER TYPE 부담.
- 약관 id 만 사용 → 위 문제.

**트레이드오프**
- VARCHAR(50) → kebab-case 표준 (`marketing-consent`, `privacy-policy`). 50 char 충분.

---

### 3.2 terms.`version VARCHAR(20)`

**왜 필요한가**
- 같은 약관의 변경 이력. 사용자가 동의한 정확한 텍스트 입증.
- `(code, version) UNIQUE` 로 중복 등록 방지.

**왜 semver-like (v1.0, v1.1) 권장**
- v1.0 → v1.1 (minor 변경, 자동 동의 OK)
- v1.x → v2.0 (major 변경, 재동의 필수)
- 정책 분기 명확.

**안 하면 무슨 문제**
- 버전 없이 date 만 (`effective_at`) 로 식별 → 같은 날 두 번 변경 시 혼동.
- application 의 "어떤 버전에 동의했는지" 추적 부정확.

**대안과 왜 안 됨**
- date 만 사용 → 위 문제. semver 보다 명확성 ↓.
- INT (1, 2, 3...) → 의미 없음. v1.0 / v2.0 가 사람이 읽기 쉬움.

**트레이드오프**
- 버전 명명 컨벤션 정책 필요 (legal 팀과 정의). minor / major 의 기준.

---

### 3.3 terms.`required BOOLEAN`

**왜 필요한가**
- 필수 = 가입 시 강제 동의 (서비스 / 개인정보 / 만 14세).
- 선택 = 마케팅 / 위치 — 사용자 결정.
- application 의 "동의 체크 모두 안 됐는데 가입 진행" 거부 로직 분기.

**안 하면 무슨 문제**
- 필수/선택 구분 X → application 의 검증 로직이 hardcoded ("service-terms 는 필수"). 새 필수 약관 추가 시 코드 변경.
- 한국 정보통신망법 §50 — 마케팅 수신 동의는 **별도 동의 박스 필수** (필수 + 선택 한 박스에 묶으면 위반).

**대안과 왜 안 됨**
- application config 로 "이 code 는 필수" 매핑 → DB 와 application 이 동기화 안 됨. 운영 시 헷갈림.

**트레이드오프**
- required 변경 (선택 → 필수, 또는 반대) 시 — 옛 동의자 처리 정책 필요.

---

### 3.4 terms.`effective_at TIMESTAMPTZ`

**왜 필요한가**
- 약관 시행 시점. 미래 시점 가능 (7일 후 발효 등).
- 다중 버전 — 가장 최근 effective + use_yn=Y 가 현재 활성.

**왜 created_at 과 분리**
- created_at = DB 에 row 가 들어온 시점 (운영자 작업 시점).
- effective_at = 법적 효력 발생 시점 (배포 후 7일 cooldown 등).
- 둘이 다른 경우가 빈번 — 약관 변경은 보통 사전 공지 7일 후 발효.

**안 하면 무슨 문제**
- effective_at 없으면 created_at 만 → "오늘 등록한 약관이 즉시 발효" — 사용자 동의 없이 효력 발생 = 법적 무효.
- 사전 공지 기간 강제 정책 적용 불가.

**대안과 왜 안 됨**
- application 단 처리 → DB 만 보면 어떤 약관이 활성인지 모름. SQL 조회 / audit 비용.

**트레이드오프**
- timezone aware (TIMESTAMPTZ) 필수. KST/UTC 혼용 시 사고.

---

### 3.5 terms.`use_yn VARCHAR(1)` — 'Y'/'N'

**왜 필요한가**
- 폐기된 약관도 row 보존 (audit). 단순 삭제 X.
- 'N' = legacy = 검색에서 제외 (`WHERE use_yn = 'Y'`).

**왜 BOOLEAN 아니고 VARCHAR(1)**
- 한국 SI / 금융 컨벤션 — `Y/N` 로 표기 (BOOLEAN 보다 SQL 가독성 ↑).
- 본 vault 의 다른 도메인과 일관성. 신규 프로젝트면 BOOLEAN 도 OK.

**안 하면 무슨 문제**
- 폐기 후 row 삭제 → 옛 동의 history 의 terms_id 가 dangling FK (실은 RESTRICT 로 막힘 → 삭제 불가 → 더 큰 문제).
- 폐기 표시 없으면 → application 이 옛 / 새 버전 구분 X.

**대안과 왜 안 됨**
- `deleted_at` timestamp → audit 더 풍부하지만 본 vault 정책은 Y/N 단순.

**트레이드오프**
- 'Y'/'N' 외 값 통과 가능 (CHECK 안 걸음). 운영 정책으로 enforce.

---

### 3.6 consent_history.`consent_yn` + `agreed_at`

**왜 필요한가 (consent_yn)**
- 'Y' = 동의, 'N' = 거부.
- 필수 약관도 row 가 있어야 (가입 자체가 동의 의미라도) audit 가능.

**왜 필수 약관도 row INSERT**
- "이 사람이 정확히 언제 어떻게 동의했는지" 입증 필수.
- 가입 시점 = 동의 시점 = 분쟁 시 핵심 증거.
- "묵시적 동의" 는 법적으로 약함 — 명시적 row 가 안전.

**왜 agreed_at TIMESTAMPTZ**
- 동의 시점 — 분쟁 시 분 단위 입증.
- timezone aware — 글로벌 user 의 동의 시점 정확성.

**안 하면 무슨 문제**
- agreed_at 없이 created_at 만 → audit 부실.
- 'N' row 없이 'Y' 만 저장 → 철회 시 row 삭제 → "동의했었다는 사실" 도 사라짐 = 분쟁 시 disadvantage.

---

### 3.7 consent_history.`ip_address` / `user_agent`

**왜 필요한가**
- 동의의 진위 검증. "이 동의가 정말 user 본인의 행위였나" 입증.
- 분쟁 시 "다른 사람이 user 명의로 동의했다" 주장 반박.
- 의심스러운 동의 (같은 IP 에서 여러 user 동시 동의) 감지.

**안 하면 무슨 문제**
- 분쟁 시 입증 자료 부실 — "user 가 직접 동의했나 / 다른 사람이 했나" 답 X.
- 봇 / 자동 가입의 패턴 분석 불가.

**대안과 왜 안 됨**
- 별도 audit_log 테이블 → join 비용. consent_history 안에 같이 두는 게 자연.

**트레이드오프**
- IP 도 PII (GDPR) → 보유 기간 정책 필요.
- 정확도 한계 — proxy / VPN 시 부정확.

---

### 3.8 왜 `ON DELETE CASCADE`

**선택지**
- (A) **CASCADE** (본 vault) — GDPR right to erasure / 한국 개인정보보호법 파기 의무.
- (B) **soft delete + 보존** — 분쟁 대비 audit (10년).
- (C) **anonymize** (user_id 만 NULL) — 중간.

**왜 CASCADE 선택**
- 사용자 탈퇴 시 PII (IP, user_agent, 동의 기록) 가 같이 사라져야 법적 안전.
- audit 자료가 필요한 경우 별도 anonymized log (user_id 제거 후 통계만) 로 분리.

**언제 (B) 가 맞는가**
- 금융 / 의료 등 강한 audit 의무 — 동의 기록 보존 의무가 파기 의무보다 우선.
- 한국 전자상거래법은 5년 보존 — terms_consent 도 적용 가능 (법적 해석 따라).

**트레이드오프**
- CASCADE → 탈퇴 즉시 동의 기록 사라짐. 탈퇴 후 분쟁 시 "동의했었다" 입증 불가.
- 본 vault: CASCADE + 옛 동의자의 anonymized stats 만 별도 보존 (옵션).

---

## 4. 인덱스 — 왜

### `ux_terms_code_version (code, version)` UNIQUE
- 같은 (code, version) 두 번 등록 방지.
- application 의 "이 버전이 이미 있나?" 검사를 DB 가 안전망.

### `ix_terms_active_code (code, effective_at DESC) WHERE use_yn = 'Y'` partial
- "현재 활성 약관" 조회 최적.
- partial — 폐기된 약관 (use_yn='N') 은 인덱스에서 제외 → 크기 ↓.
- 안 하면: 매 약관 조회 시 풀스캔 (수십 row 라도 잦은 호출 시 누적).

### `ix_user_terms_consent_user (user_id)`
- "user 의 모든 동의 history" 조회.
- /me/consents 화면 / 탈퇴 시 PII 정리.

### `ix_user_terms_consent_terms (terms_id)`
- 분석: "v2.0 에 몇 명이 동의했나" / "v1.0 동의 후 v2.0 미동의 user 누구".
- audit / CS 응대.

### `ix_user_terms_consent_user_agreed (user_id, agreed_at DESC)`
- "user 의 가장 최근 동의" — DISTINCT ON 의 효율적 인덱스 활용.

---

## 5. 시드 데이터 — 초기 약관 입력

```sql
-- V2_1__seed_initial_terms.sql
INSERT INTO terms (id, code, version, title, content, required, effective_at, use_yn) VALUES
('01HZTERMS001SERVICE', 'service-terms', 'v1.0', '서비스 이용약관',
 '<약관 내용...>', true, now(), 'Y'),
('01HZTERMS002PRIVACY', 'privacy-policy', 'v1.0', '개인정보 처리방침',
 '<처리방침 내용...>', true, now(), 'Y'),
('01HZTERMS003MARKETING', 'marketing', 'v1.0', '마케팅 정보 수신 동의',
 '<수신 동의 내용...>', false, now(), 'Y'),
('01HZTERMS004AGE', 'age-14', 'v1.0', '만 14세 이상 확인',
 '만 14세 이상입니다.', true, now(), 'Y');
```

**왜 Flyway seed 권장 (application bootstrap 아님)**
- 모든 환경 (dev, staging, prod) 에 일관된 초기 데이터.
- application bootstrap 은 race condition / 순서 이슈 발생 가능.

**왜 ID 가 hardcoded (`01HZTERMS001SERVICE`)**
- 환경 간 동일 ID — 마이그레이션 / 비교 시 유리.
- 단 ULID 의 timestamp 부분이 무의미해짐 (실제 ULID 생성이 아님). placeholder 의미만.

---

## 6. 약관 버전 변경 흐름

### 6.1 새 버전 배포 — 7일 사전 공지

```sql
-- 옛 버전 폐기 (use_yn='N')
UPDATE terms SET use_yn = 'N' WHERE code = 'privacy-policy' AND version = 'v1.0';

-- 새 버전 등록 (7일 후 발효)
INSERT INTO terms (id, code, version, title, content, required, effective_at, use_yn)
VALUES ('01HZTERMS_NEW', 'privacy-policy', 'v2.0', '...', '...', true,
        now() + INTERVAL '7 days', 'Y');
```

**왜 7일 사전 공지**
- 한국 정보통신망법 §27의2 — 약관 변경 시 **30일 전 공지** (단 사용자 유리 변경은 7일).
- 운영 정책: 변경 안내 메일 + 화면 모달 + 7~30일 cooldown.

### 6.2 옛 동의자 — 재동의 정책

| 정책 | 처리 | 법적 안전성 |
| --- | --- | --- |
| **재동의 강제** (본 vault) | 새 버전 효력 후 로그인 시 동의 화면 강제 | ✅ 최고 |
| **묵시적 계속 동의** | 안내만, 사용자가 명시적 거절 시 처리 | ⚠️ 변경 내용에 따라 |
| **마이그레이션** (자동 동의) | 옛 v1.0 동의 = 새 v2.0 자동 동의 | ❌ 위험 (법적 검토 필수) |

**왜 본 vault 는 재동의 강제**
- 가장 법적으로 안전.
- 사용자에게 "변경 내용 명시 + 명시 동의" UI 제공.
- 한국 PIPC / GDPR 모두 권장.

```sql
-- v2.0 발효 후 로그인 시
SELECT * FROM user_terms_consent_history
WHERE user_id = ? AND terms_id IN (
    SELECT id FROM terms WHERE code = 'privacy-policy' AND version = 'v2.0'
)
ORDER BY agreed_at DESC LIMIT 1;
-- 없으면 재동의 요구
```

---

## 7. 한국 개인정보보호법 핵심

### 7.1 필수 약관

| 약관 | 법적 근거 | 미동의 시 |
| --- | --- | --- |
| 서비스 이용약관 | 약관규제법 | 서비스 가입 불가 |
| 개인정보 처리방침 | 개인정보보호법 §15 | 개인정보 처리 불가 → 가입 불가 |
| 만 14세 이상 확인 | 정보통신망법 §31 | 법정대리인 동의 별도 |

### 7.2 선택 약관

| 약관 | 법적 근거 | 미동의 시 |
| --- | --- | --- |
| 마케팅 수신 동의 | 정보통신망법 §50 | 마케팅 메일 / SMS 발송 X |
| 광고성 정보 수신 | 정보통신망법 §50 | 광고 알림 X |
| 위치정보 동의 | 위치정보법 §18 | 위치 기능 X |

### 7.3 한 박스에 묶지 말 것

**잘못된 UI**
```
☐ 모든 약관에 동의합니다 (필수+선택 한 번에)
```

**올바른 UI**
```
☐ [필수] 서비스 이용약관
☐ [필수] 개인정보 처리방침
☐ [필수] 만 14세 이상입니다
☐ [선택] 마케팅 정보 수신
```

**왜 분리해야 하는가**
- 정보통신망법 §50 — 선택 동의 항목은 **별도 동의** 받아야 (한 박스 위반).
- PIPC 과징금 사례 다수 (한 박스로 묶은 사이트).

### 7.4 철회 — 사용자가 동의 취소

```sql
-- 새 row 'N' 으로 INSERT (옛 'Y' row 는 유지)
INSERT INTO user_terms_consent_history (id, user_id, terms_id, consent_yn, agreed_at)
VALUES (?, ?, ?, 'N', now());
```

**"현재 동의 상태" 조회**

```sql
SELECT DISTINCT ON (terms_id) terms_id, consent_yn, agreed_at
FROM user_terms_consent_history
WHERE user_id = ?
ORDER BY terms_id, agreed_at DESC;
```

**왜 새 row INSERT (UPDATE 아님)**
- audit history — 동의 → 철회 → 재동의 흐름 보존.
- 분쟁 시 "이 사용자가 정확히 언제 어떤 상태였는지" 입증.

---

## 8. JPA Entity

```java
@Entity
@Table(name = "terms")
public class TermsJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(nullable = false, length = 50) private String code;
    @Column(nullable = false, length = 20) private String version;
    @Column(nullable = false, length = 200) private String title;
    @Column(nullable = false, columnDefinition = "TEXT") private String content;
    @Column(nullable = false) private boolean required;
    @Column(name = "effective_at", nullable = false) private Instant effectiveAt;
    @Column(name = "use_yn", nullable = false, length = 1) private String useYn = "Y";
    @Column(name = "created_at", nullable = false) private Instant createdAt;
}

@Entity
@Table(name = "user_terms_consent_history")
public class UserTermsConsentJpaEntity {
    @Id @Column(length = 26) private String id;
    @Column(name = "user_id", nullable = false, length = 26) private String userId;
    @Column(name = "terms_id", nullable = false, length = 26) private String termsId;
    @Column(name = "consent_yn", nullable = false, length = 1) private String consentYn;
    @Column(name = "agreed_at", nullable = false) private Instant agreedAt;
    @Column(name = "ip_address", length = 45) private String ipAddress;
    @Column(name = "user_agent", length = 255) private String userAgent;
}
```

> 💡 `@ManyToOne` (Terms 와 association) 안 만든 이유 — Aggregate 경계. terms_id 만 String 으로. [[../domain-model/aggregate-boundaries#3]].

---

## 9. 조회 패턴

| 패턴 | SQL | 빈도 | 인덱스 |
| --- | --- | --- | --- |
| 현재 활성 약관 모두 (가입 화면) | `WHERE use_yn = 'Y' AND effective_at <= now() ORDER BY code, effective_at DESC` | 매 가입 시도 | `ix_terms_active_code` |
| 특정 code 의 현재 버전 | `WHERE code = ? AND use_yn = 'Y' AND effective_at <= now() ORDER BY effective_at DESC LIMIT 1` | 변경 감지 | `ix_terms_active_code` |
| 특정 user 의 현재 동의 상태 | DISTINCT ON (terms_id) (위 §7.4) | /me/consents | `ix_user_terms_consent_user_agreed` |
| Audit — 특정 시점의 동의 상태 | `WHERE user_id = ? AND agreed_at <= ?` | 분쟁 / CS | `ix_user_terms_consent_user_agreed` |

---

## 10. 함정 모음 — "이걸 안 하면 X 가 터짐"

### 함정 1 — `users.marketing_agreed BOOLEAN`
버전 변경 / 철회 추적 X. 새 약관 배포 시 마이그레이션 폭탄.
→ 별도 history table.

### 함정 2 — 필수 / 선택 한 box 에 묶기
한국 정보통신망법 §50 위반. PIPC 과징금 사례 다수.
→ UI 에서 별도 체크박스 + DB 의 `required` flag 로 검증.

### 함정 3 — 옛 약관 row 삭제
audit 무용지물 — 옛 동의자의 동의 사실 자체가 사라짐.
→ `use_yn='N'` 만. row 보존.

### 함정 4 — `(code, version) UNIQUE` 누락
같은 version 두 번 등록 가능 → 어떤 게 정본인지 혼동. application 매핑 오류.
→ UNIQUE 강제.

### 함정 5 — 동의 시점 (`agreed_at`) 안 저장
분쟁 시 "언제 동의했는지" 증명 X → 법적 disadvantage.
→ TIMESTAMPTZ 필수.

### 함정 6 — IP / user_agent 안 저장
"이 동의가 user 본인의 행위였나" 입증 X. 봇 / 도용 시도 분석 불가.
→ 가급적 저장 + GDPR 보유 기간 정책.

### 함정 7 — 약관 버전 변경 시 옛 동의자 자동 마이그레이션
법적 위험 — "user 모르게 새 약관에 동의" 처리 = 효력 없음.
→ 재동의 요구가 안전.

### 함정 8 — 약관 폐기 (`use_yn='N'`) 후 동의 row 도 삭제
audit 깨짐. 옛 동의자의 동의 사실 자체가 없어짐.
→ row 는 보존, terms 의 use_yn 만 'N'.

### 함정 9 — 회원가입 시 필수 약관 동의 안 검증
필수 약관 미동의로 가입 가능 → 법적 위반.
→ Bean Validation `@AssertTrue` + 도메인 검증 + DB CHECK 3중.

### 함정 10 — 약관 content 가 HTML 인데 escape 안 함
관리자가 입력한 HTML 이 사용자에 그대로 → XSS.
→ OWASP Java HTML Sanitizer (출력 시) + 저장 시 sanitize 옵션.

### 함정 11 — 7일 사전 공지 안 함
약관 변경 즉시 효력 → 법적 효력 없음 (사용자 모르게 변경).
→ effective_at = now() + 7일.

### 함정 12 — TIMESTAMP (without tz)
서버 timezone 변경 시 모든 의미 깨짐. 글로벌 운영 시 재앙.
→ TIMESTAMPTZ 필수.

---

## 11. 관련

- [[database|↑ database hub]]
- [[users-table]] — `user_id` FK
- [[../design-decisions#10 약관]] — 정책 결정 (재동의 vs 마이그레이션)
- [[../signup-impl#6 UserTermsConsentService]] — application 코드
- [[../domain-model/aggregate-boundaries#3]] — 왜 User 의 일부가 아닌가
- 외부 — 한국 개인정보보호법 §15, 정보통신망법 §31/§50
