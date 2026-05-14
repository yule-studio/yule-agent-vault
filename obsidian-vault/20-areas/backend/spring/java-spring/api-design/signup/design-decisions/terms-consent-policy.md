---
title: "약관 / 개인정보 동의 정책 — 한국 특화"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:45:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - terms
  - consent
  - pipc
---

# 약관 / 개인정보 동의 정책 — 한국 특화

**[[design-decisions|↑ design-decisions hub]]**

> "약관 / 개인정보 동의를 어떻게 받고 관리하나" — 한국 SaaS 의 법적 필수.
> 정책이 부실하면 **PIPC 과징금 + 분쟁 시 입증 불가**.

---

## 1. 본 vault 결정

- **별도 테이블 + 버전 관리** (`terms` + `user_terms_consent_history`).
- **필수 / 선택 명확 분리** (한 박스 묶기 금지).
- **버전 변경 시 재동의** (자동 마이그레이션 X).
- **30일 사전 공지** (한국 정보통신망법).

자세히: [[../database/terms-tables]].

---

## 2. 4구조 — 정책 결정

### 2.1 왜 별도 테이블 (boolean 컬럼 아님)

**왜 필요한가**
- 약관 버전 변경 추적 — v1.0 → v2.0 → ...
- 동의 history (동의 → 철회 → 재동의) — 분쟁 시 입증.
- 다중 약관 (서비스 / 개인정보 / 마케팅 / 위치 / 광고 / 만 14세) — 컬럼 N개 폭증 회피.

**안 하면 무슨 문제 (boolean 컬럼만)**
- 약관 변경 시 옛 동의자 처리 X.
- "언제 동의했나" audit X → 분쟁 시 disadvantage.
- 새 약관 추가 시 ALTER TABLE 빈번.

**대안과 왜 안 됨**
- JSON 컬럼 (`users.consents JSONB`) → 검색 / 인덱싱 / audit 부실.

**트레이드오프**
- 테이블 2개 + join 부담. 동의 history 가 매 가입마다 N row.

---

### 2.2 왜 필수 / 선택 명확 분리

**왜 필요한가 (법적)**
- 한국 정보통신망법 §50 — 마케팅 수신 동의는 **별도 동의** 받아야.
- 한 박스에 묶기 = 위반 + PIPC 과징금.

**구체적 사례**
- 인터파크 (2020) — 한 박스 묶기로 과징금.
- 다수 SaaS — 마케팅 + 필수 통합 동의로 시정 명령.

**안 하면 무슨 문제**
- 한국 PIPC 시정 조치 + 과징금.
- 사용자가 "모르고 마케팅 동의" 클레임 / 민원.

**대안과 왜 안 됨**
- "모든 약관 동의" 단일 체크 → 위반.

**구현 강제**
- DB 의 `terms.required BOOLEAN` 컬럼.
- UI 가 required 별 박스 분리.
- 가입 API 가 필수 약관 동의 검증.

---

### 2.3 왜 버전 변경 시 재동의 (자동 마이그레이션 X)

**왜 필요한가**
- v1.0 → v2.0 의 내용 변경 → 사용자가 새 약관에 명시 동의해야 효력.
- 자동 마이그레이션 = 묵시적 동의 = 법적 효력 약함.

**한국 정보통신망법 §27의2**
- 약관 변경 시 30일 전 공지.
- 사용자 유리 변경은 7일.

**안 하면 무슨 문제**
- 분쟁 시 "새 약관에 동의 안 했음" → 효력 무효.

**대안과 왜 안 됨**
- 자동 마이그레이션 → 법적 위험.
- "안내만, 이용 = 동의" → 모호함. 명시 동의가 안전.

**구현**
```sql
-- v2.0 발효 후 로그인 시
SELECT * FROM user_terms_consent_history
WHERE user_id = ? AND terms_id = (
    SELECT id FROM terms WHERE code = 'privacy-policy' AND version = 'v2.0'
);
-- 없으면 재동의 화면
```

---

### 2.4 왜 30일 사전 공지

**한국 정보통신망법 §27의2**
- 약관 변경 시 30일 전 공지 (사용자 유리 = 7일).
- 미준수 시 변경 효력 발생 X.

**구현**
```sql
INSERT INTO terms (id, code, version, content, effective_at, use_yn)
VALUES (?, 'privacy-policy', 'v2.0', '...', now() + INTERVAL '30 days', 'Y');
```

→ effective_at = 30일 후. 그 사이 안내 메일 / 화면 모달.

**안 하면 무슨 문제**
- PIPC 시정 조치.
- 분쟁 시 변경 효력 인정 X.

---

## 3. 필수 / 선택 약관 — 한국 법적 분류

### 3.1 필수 (가입 자체 조건)

| 약관 | 법적 근거 | 미동의 시 |
| --- | --- | --- |
| 서비스 이용약관 | 약관규제법 | 서비스 가입 불가 |
| 개인정보 처리방침 | 개인정보보호법 §15 | 개인정보 처리 불가 |
| 만 14세 이상 확인 | 정보통신망법 §31 | 법정대리인 동의 별도 흐름 |

### 3.2 선택 (사용자 결정)

| 약관 | 법적 근거 | 미동의 시 |
| --- | --- | --- |
| 마케팅 정보 수신 (이메일) | 정보통신망법 §50 | 마케팅 메일 발송 X |
| 마케팅 정보 수신 (SMS) | 정보통신망법 §50 | 마케팅 SMS 발송 X |
| 위치정보 수집 | 위치정보법 §18 | 위치 기능 X |
| 광고성 정보 수신 | 정보통신망법 §50 | 광고 알림 X |

---

## 4. UI 패턴

### 4.1 올바른 UI

```
☐ [필수] 서비스 이용약관 동의           [전문 보기]
☐ [필수] 개인정보 처리방침 동의          [전문 보기]
☐ [필수] 만 14세 이상입니다              

☐ [선택] 마케팅 정보 수신 동의 (이메일)
☐ [선택] 마케팅 정보 수신 동의 (SMS)

[모든 필수 약관에 동의]   [모든 선택 약관에 동의]
```

**왜 별도 [모든 필수] 버튼**
- 사용자 편의 — 필수만 빠른 동의.
- 선택과 분리 = 법적 안전.

### 4.2 잘못된 UI

```
☐ 모든 약관에 동의합니다 (필수+선택 한 번에)        ❌ 정보통신망법 §50 위반
```

---

## 5. 14세 미만 — 법정대리인 동의 흐름

```
[가입 화면]
  사용자가 생년월일 입력 → 만 14세 미만 감지
   ↓
[법정대리인 정보]
  부모 / 보호자 이름 + 휴대폰
   ↓
[법정대리인 휴대폰 인증]
  부모 휴대폰에 SMS — 동의 링크
   ↓
[법정대리인 약관 동의]
  부모 명의로 약관 동의 (별도 row)
   ↓
[가입 완료]
  user row INSERT (status=ACTIVE, parent_consent_id=?)
```

**왜 필요**
- 정보통신망법 §31 — 만 14세 미만은 법정대리인 동의 필수.
- 미준수 시 사고 발생 시 책임 ↑.

**언제 안 받음**
- B2B SaaS — 사용자가 모두 성인 가정.
- 게임 — PASS 본인인증으로 연령 확인.

---

## 6. 철회 흐름

```sql
-- 새 row 'N' 으로 INSERT (옛 'Y' row 는 유지)
INSERT INTO user_terms_consent_history (id, user_id, terms_id, consent_yn, agreed_at)
VALUES (?, ?, ?, 'N', now());
```

**왜 새 row INSERT (UPDATE 아님)**
- 동의 → 철회 → 재동의 흐름 audit.
- 분쟁 시 "이 사용자가 정확히 언제 어떤 상태였는지" 입증.

**철회 후 효과**
- 마케팅 메일 / SMS 발송 stop (application 단 검증).
- 옛 동의 기록은 보존 (audit).

---

## 7. 약관 폐기 정책

```sql
UPDATE terms SET use_yn = 'N' WHERE id = ?;
```

**왜 row 삭제 X**
- 옛 동의 history 가 terms_id 참조 → FK 깨짐.
- 분쟁 시 "옛 약관 내용" 입증 X.

**보유 기간**
- 옛 약관 row — 5년+ (계약 / 회계 보존 의무).
- 옛 동의 history — 5년+ (분쟁 대비).

---

## 8. 함정 모음

### 함정 1 — 한 박스 묶기
정보통신망법 §50 위반 + PIPC 과징금.
→ 필수 / 선택 명확 분리.

### 함정 2 — boolean 컬럼 (`users.marketing_agreed`)
버전 / 철회 추적 X.
→ 별도 테이블.

### 함정 3 — 자동 마이그레이션 (옛 동의 = 새 동의)
법적 위험.
→ 재동의 강제.

### 함정 4 — 사전 공지 없이 변경
정보통신망법 §27의2 위반.
→ 30일 (사용자 유리는 7일).

### 함정 5 — 옛 약관 row 삭제
audit / 분쟁 입증 X.
→ use_yn='N' 만.

### 함정 6 — 동의 시점 (agreed_at) 미저장
"언제 동의했나" 입증 X.
→ TIMESTAMPTZ + IP / user_agent.

### 함정 7 — IP / user_agent 미저장
"본인이 했나" 입증 X. 봇 / 도용 분석 불가.
→ 저장 + GDPR 보유 기간 정책.

### 함정 8 — 14세 미만 동의 흐름 없음
정보통신망법 §31 위반.
→ 법정대리인 동의 흐름.

### 함정 9 — 마케팅 동의 안 받고 발송
정보통신망법 §50 위반.
→ application 의 발송 시 동의 검증.

### 함정 10 — 동의 철회 시 row UPDATE (history 안 남김)
audit 깨짐.
→ 새 row INSERT.

### 함정 11 — 가입 후 약관 동의 별도 화면
사용자가 모르고 가입 완료.
→ 가입 흐름 안에서 동의.

### 함정 12 — content 의 HTML XSS
관리자가 입력한 HTML 그대로 → XSS.
→ OWASP HTML Sanitizer.

---

## 9. 다른 컨텍스트

### 9.1 글로벌 (GDPR)

```yaml
terms-consent:
  granularity: explicit + per-purpose
  data-portability: yes (사용자 데이터 export)
  right-to-erasure: yes (탈퇴 30일 후 완전 파기)
```

### 9.2 미국 (CCPA)

```yaml
terms-consent:
  primary: opt-out (selling personal data)
  reason: 캘리포니아주법
```

### 9.3 B2B SaaS

```yaml
terms-consent:
  primary: master-service-agreement (계약서)
  per-user: 간소화
```

---

## 10. 관련

- [[design-decisions|↑ hub]]
- [[../database/terms-tables]] — schema + 14 함정
- [[../signup-impl#6]] — UserTermsConsentService 구현
- 외부 — 한국 개인정보보호법, 정보통신망법 §27/§31/§50, GDPR
