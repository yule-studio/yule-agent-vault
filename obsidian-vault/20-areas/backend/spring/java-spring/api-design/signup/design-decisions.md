---
title: "auth §4 — 설계 의사결정 + 권장 도구 가이드"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:05:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - tooling
---

# auth §4 — 설계 의사결정 + 권장 도구 가이드

**[[signup|↑ hub]]**  ·  ← [[requirements]]  ·  → [[database]]

> ⭐ **이 폴더의 핵심 결정 가이드.** 어떤 도구 / 알고리즘 / 정책을 골라야 하나 — 각 항목별 권장 + trade-off.

---

## 1. 의사결정 체크리스트 (먼저 정해야 할 7개)

| # | 결정 | 옵션 | 본 vault 기본 |
| --- | --- | --- | --- |
| 1 | 가입 인증 채널 | 이메일 / 휴대폰 / 둘 다 | **둘 다 가능, 휴대폰 우선** (한국 SaaS) |
| 2 | 패스워드 해시 | Argon2id / bcrypt / scrypt | **Argon2id** |
| 3 | 토큰 모델 | JWT / Session / 둘 다 | **JWT** (access + opaque refresh) |
| 4 | refresh 저장소 | RDB / Redis | **RDB** 시작 → Redis 마이그레이션 |
| 5 | 이메일 발송 도구 | SES / SendGrid / SMTP / Mailgun / Postmark | **AWS SES** |
| 6 | SMS 발송 도구 | NCP SENS / AlimTalk / Twilio / 알리고 | **NCP SENS + AlimTalk** (한국) |
| 7 | 소셜 로그인 | Apple / Google / Kakao / Naver | **4개 다** (한국 SaaS 표준) |

---

## 2. 가입 인증 채널 — 이메일 vs 휴대폰 vs 둘 다

| 채널 | 적합 | 비고 |
| --- | --- | --- |
| **이메일만** | B2B / 글로벌 SaaS / 데스크탑 중심 | Notion / Slack / Github |
| **휴대폰만** | 한국 모바일 우선 / 보안 critical | 카카오 / 당근 / 무신사 |
| **둘 다 (이메일 ID + 휴대폰 인증)** | 한국 SaaS 표준 | 휴대폰 인증으로 본인 확인 + 이메일 ID 로 로그인 |
| **둘 다 (이메일 + 휴대폰 ID 가능)** | 유연 | 사용자 선택 |

**본 vault 정책**: **이메일 ID + 휴대폰 인증** 표준. 추후 사업 도메인 별 조정.

### 휴대폰 인증의 종류

| 방식 | 신뢰도 | 비용 | 사용처 |
| --- | --- | --- | --- |
| **SMS 인증번호 (6자리)** | 중간 | 저렴 | 일반 가입 |
| **AlimTalk** | 중간 | SMS 보다 저렴 (≈10원 vs 30원) | 카카오톡 사용자 대상 |
| **본인인증 (PASS / NICE)** | **높음 (실명)** | 비쌈 (~200원) | 금융 / 의료 / 청소년보호 |
| **ARS** | 높음 (음성) | 비쌈 | 시각장애인 / 노약자 |

**본 vault 정책**: 일반은 **SMS** 또는 **AlimTalk**. 금융 / 본인확인 필요는 **PASS**.

---

## 3. 패스워드 해시 알고리즘

| 알고리즘 | 권장 (2026) | 비고 |
| --- | --- | --- |
| **Argon2id** | ✅ OWASP 1순위 | m=64MB, t=3, p=4 |
| **bcrypt** | ✅ 대안 | cost ≥ 12. 72-byte 입력 제한 |
| scrypt | △ | argon2 후순위 |
| PBKDF2-HMAC-SHA256 | △ | iterations ≥ 600,000 |
| MD5 / SHA-1 / SHA-256 단독 | ❌❌❌ | **사고**. 초당 수십억 시도 가능 |

### Argon2id 파라미터 결정

| 사용처 | m (KB) | t | p |
| --- | --- | --- | --- |
| 일반 서버 (인텔 4core, 2026) | **65536 (64MB)** | **3** | **4** |
| 작은 서버 / Lambda | 16384 (16MB) | 3 | 1 |
| 모바일 / IoT | 4096 (4MB) | 3 | 1 |

**본 vault 기본**: 일반 서버 — 인증 시 100~300ms 가 일반.

→ 코드: [[security#4 알고리즘 선정]].

---

## 4. 토큰 모델 — JWT vs Session

| | JWT (Stateless) | Session (Stateful) | Hybrid |
| --- | --- | --- | --- |
| 저장 | 클라 (LocalStorage / Cookie) | 서버 (Redis / DB) | 둘 다 |
| 검증 | 서명만 (DB 호출 X) | 매 요청 DB lookup | 둘 다 |
| 무효화 | 어려움 (만료까지) | 즉시 가능 | 즉시 가능 |
| 분산 | 자연스러움 | sticky session 또는 공유 store | 공유 store |
| 사용처 | MSA / API 표준 | 모놀리식 / 강한 invalidation | 큰 SaaS |

**본 vault**: **JWT 표준** + 다음 2 보완:
- access 짧게 (15분) — 무효화 부담 ↓
- refresh 는 opaque + 서버 매핑 — 즉시 무효 가능 (DB delete)

### JWT 서명 알고리즘

| 알고리즘 | 키 | 사용처 |
| --- | --- | --- |
| **HS256** | 대칭 (single secret) | 모놀리식 / 단일 서비스 |
| **RS256** | 비대칭 (private + public) | MSA / 외부 검증자 |
| **ES256** | ECDSA P-256 | RS256 의 가벼운 대안 |
| none / weak HS256 | — | ❌ 절대 X |

**본 vault**: **HS256** (모놀리식 가정). MSA 면 **RS256** + JWKS.

---

## 5. 이메일 발송 도구

| 도구 | 비용 (1k 메일) | 배달율 | 속도 | 한국 인프라 | 본 vault 권장 |
| --- | --- | --- | --- | --- | --- |
| **AWS SES** | $0.10 | 좋음 | 좋음 | (서울 리전) | ✅ 1순위 (비용 + 안정) |
| **SendGrid** | $20 (40k 무료) | 매우 좋음 | 빠름 | 좋음 | 2순위 (배달율 critical 시) |
| **Mailgun** | $35 | 좋음 | 좋음 | 좋음 | — |
| **Postmark** | $15 | 매우 좋음 (트랜잭션 mail 특화) | 빠름 | 좋음 | 가입 / 결제 같은 critical mail |
| 자체 SMTP | $0 (서버 비용만) | ⚠️ Gmail / Naver 가 spam 처리 | 보통 | 좋음 | 비추 |
| **NHN Cloud Email** | 저렴 | 좋음 (국내) | 좋음 | ✅ | 국내 한정 사용 |

**본 vault 기본**: **AWS SES** (서울 리전, sandbox 해제 후 사용).

### 발송 시 주의

- **SPF / DKIM / DMARC** DNS 설정 — 안 하면 spam 폴더 직행
- 첫 송신은 IP warming (소량씩 늘리기)
- bounce / complaint webhook 처리 (수신자 제거)
- HTML + text 둘 다 (text 만 받는 클라 대응)

코드: [[email-verification-impl]].

---

## 6. SMS 발송 도구 (국내)

| 도구 | 비용 (건) | 신뢰도 | 한국 인프라 | 본 vault 권장 |
| --- | --- | --- | --- | --- |
| **NCP SENS (네이버)** | 9~13원 | 매우 좋음 | ✅ (국내 1위) | ✅ 1순위 |
| **AlimTalk (카카오)** | 7~12원 | 매우 좋음 | ✅ (카카오 사용자) | ✅ 2순위 (조합) |
| **Twilio** | 50원+ | 좋음 (글로벌) | 글로벌 | 해외 사용자 대상 |
| **알리고** | 8~13원 | 좋음 | ✅ | 대안 |
| **Daou (다오) SMS API** | 10원 | 좋음 | ✅ | 대안 |
| **NHN Cloud SMS** | 13원 | 좋음 | ✅ | 대안 |

**본 vault 기본**: **NCP SENS** + 카카오 사용자는 **AlimTalk** fallback. 미수신 시 SMS 재발송 (3분 후).

### SMS 인증 흐름

```
POST /auth/verify/phone/request   { phone: "010-0000-0000" }
   ↓
서버: 6자리 코드 생성 + Redis 저장 (TTL 3분) + NCP SENS API 호출
   ↓
사용자 휴대폰에 SMS
   ↓
POST /auth/verify/phone/confirm   { phone, code }
   ↓
서버: Redis lookup + 일치 + verifiedAt 마킹 (TTL 10분 — signup 진행 동안)
   ↓
signup 요청에 phoneAuthToken 포함 → 검증 통과
```

상세: [[phone-verification-impl]].

### 휴대폰 본인인증 (PASS / NICE) — 금융·의료

| 도구 | 본인인증 종류 | 비용 | 비고 |
| --- | --- | --- | --- |
| **PASS** | 휴대폰 본인인증 | ~150원 | 통신 3사 통합 |
| **NICE 평가정보** | 휴대폰 / 신용카드 / 공동인증 | ~200원 | 가장 범용 |
| **KCB** | 휴대폰 본인인증 | ~150원 | NICE 대안 |
| **SCI (서울크레딧평정원)** | 본인확인 | ~250원 | 전자서명 가능 |

→ 금융 / 의료 / 청소년보호 / 게임 (실명 인증 의무) 도메인. 일반 SaaS 는 SMS 로 충분.

---

## 7. 소셜 로그인 — Apple / Google / Kakao / Naver

| Provider | 한국 시장 점유 | 구현 난이도 | 토큰 검증 |
| --- | --- | --- | --- |
| **Kakao** | 매우 높음 (3,500만+) | 쉬움 | `/v2/user/me` + access_token |
| **Naver** | 높음 | 쉬움 | `/v1/nid/me` + access_token |
| **Google** | 보통 | 쉬움 | `tokeninfo` + id_token |
| **Apple** | iOS 사용자 (필수) | **어려움** (JWT JWKS + 첫 로그인 만 email) | id_token JWT 검증 |

**본 vault 기본**: 한국 SaaS 면 **Kakao + Naver** 우선. iOS 앱이면 **Apple 필수** (App Store 정책). Google 은 옵션.

상세: [[../oauth2-social-login]].

---

## 8. 2FA — 도입 시점

| 단계 | 시점 |
| --- | --- |
| 가입 시 강제 | ❌ — 가입 friction 폭증 |
| 로그인 시 옵션 | ✅ — 사용자가 설정에서 활성 |
| **민감 작업** (패스워드 변경 / 결제 / 인출) 시 강제 | ✅ — step-up auth |
| 관리자 / VIP 강제 | ✅ — 본 vault 권장 |

**본 vault 기본**: **옵션 + step-up**. ADMIN role 은 강제.

상세: [[../two-factor-auth]].

---

## 9. refresh 저장소 — RDB vs Redis

| | RDB (PostgreSQL) | Redis |
| --- | --- | --- |
| 영속 | ✅ | TTL 자동 만료 |
| latency | ms | μs |
| 검색 가능 (audit) | ✅ | ⚠️ |
| 비용 | 무료 (이미 있는 DB) | Redis 추가 운영 |
| 사용처 | 중소 SaaS / audit 중시 | 대형 SaaS / latency critical |

**본 vault**: **RDB 시작** → 성장 후 Redis. 동시 사용도 OK (RDB 가 진실, Redis 캐시).

---

## 10. 약관 / 개인정보 동의 — 한국 특화

### 필수 요소 (개인정보보호법 / 정보통신망법)

| 약관 | 필수 | 비고 |
| --- | --- | --- |
| 서비스 이용약관 | ✅ | 가입 필수 |
| 개인정보 처리방침 | ✅ | 가입 필수 |
| 만 14세 이상 확인 | ✅ (한국) | 또는 법정대리인 동의 |
| 마케팅 수신 동의 | △ | 선택 |
| 위치정보 이용약관 | △ (위치 서비스만) | 당근식 |
| 광고성 정보 수신 (이메일 / SMS) | △ | 선택 별도 |

### 저장 방식

```sql
-- 약관 자체 (버전 관리)
CREATE TABLE terms (
    id           CHAR(26) PRIMARY KEY,
    code         VARCHAR(50) NOT NULL,           -- 'service-terms' / 'privacy-policy'
    version      VARCHAR(20) NOT NULL,
    title        VARCHAR(200) NOT NULL,
    content      TEXT NOT NULL,
    required     BOOLEAN NOT NULL,
    effective_at TIMESTAMPTZ NOT NULL,
    use_yn       VARCHAR(1) NOT NULL DEFAULT 'Y'
);

-- 동의 history
CREATE TABLE user_terms_consent_history (
    id         CHAR(26) PRIMARY KEY,
    user_id    CHAR(26) NOT NULL,
    terms_id   CHAR(26) NOT NULL,
    consent_yn VARCHAR(1) NOT NULL,
    agreed_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

→ **단순 `users.marketing_agreed BOOLEAN`** 으로 두지 말 것. 버전 변경 / 재동의 / 철회 추적 불가 (GDPR / 한국 개정 정책 위반).

상세: [[database#2.2]].

---

## 11. enumeration 정책

| 정책 | UX | 보안 | 본 vault |
| --- | --- | --- | --- |
| 직관적 (409 + "이미 가입된 이메일") | ✅ | ⚠️ 가입자 enumeration 가능 | 일반 SaaS |
| **차단** (항상 200 + "이메일 확인하세요") | ⚠️ 친절도 ↓ | ✅ | 금융 / 의료 |

본 vault: **직관적** (기본). 금융 도메인은 차단.

---

## 12. Idempotency-Key 정책

| 정책 | 영향 |
| --- | --- |
| 없음 | 네트워크 retry 시 중복 가입 가능 |
| Optional header | 클라가 보내면 멱등 |
| Required (모든 mutating endpoint) | 안전 + 클라 구현 부담 |

**본 vault**: **Optional**. 결제 / 주문 같은 비용 큰 endpoint 는 Required ([[../payment-pg]] / [[../order-stock]]).

---

## 13. 권장 dependency (build.gradle.kts)

```kotlin
// 패스워드 해시 (Argon2id)
implementation("com.password4j:password4j:1.8.2")

// JWT
implementation("io.jsonwebtoken:jjwt-api:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")

// 이메일
implementation("org.springframework.boot:spring-boot-starter-mail")
implementation("software.amazon.awssdk:ses:2.25.50")          // AWS SES

// SMS — NCP SENS (직접 호출)
implementation("org.springframework.boot:spring-boot-starter-webflux")  // WebClient

// 소셜 — Apple JWT 검증
implementation("com.nimbusds:nimbus-jose-jwt:9.40")

// 2FA TOTP
implementation("com.warrenstrange:googleauth:1.5.0")
implementation("com.google.zxing:core:3.5.3")                  // QR

// Rate limit
implementation("com.bucket4j:bucket4j-redis:8.10.1")
```

---

## 14. 의사결정 결과 — 본 vault 의 기본 stack

```yaml
auth:
  password-hash: Argon2id (m=64MB, t=3, p=4)
  jwt:
    algorithm: HS256
    access-ttl: 15m
    refresh-ttl: 14d
    refresh-rotation: true
    refresh-storage: RDB

  signup-verification:
    channels: [email, phone]                  # 한국 SaaS 기본
    email-provider: aws-ses
    sms-provider: ncp-sens                     # AlimTalk 옵션

  social-login:
    providers: [kakao, naver, google, apple]   # 한국 SaaS

  two-factor-auth:
    enabled: optional + admin-required
    method: totp

  password-policy:
    min-length: 8
    max-length: 128
    pwned-check: true (haveibeenpwned API)

  rate-limit:
    signup: 3/hour per IP
    login: 5/min per IP+email
    sms-request: 5/hour per phone

  enumeration: explicit (일반 SaaS) | obscured (금융)

  idempotency: optional
```

→ **이 설정에서 시작 + 도메인 별 조정** 권장.

---

## 15. 관련

- [[signup|↑ hub]]
- [[requirements]] — 이전 (§3)
- [[database]] — 다음 (§5)
- [[security]] — 보안 정책 깊이
- [[email-verification-impl]] / [[phone-verification-impl]] — 구현
- [[../oauth2-social-login]] / [[../two-factor-auth]] — 보강 레시피
