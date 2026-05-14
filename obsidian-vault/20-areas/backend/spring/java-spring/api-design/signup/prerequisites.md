---
title: "signup §0 — 전제 / 범위"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:32:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - prerequisites
---

# signup §0 — 전제 / 범위

**[[signup|↑ signup hub]]**

> 이 레시피를 적용하기 전 확인할 전제와, **언제 적용하면 좋고 / 언제 과한지** 명시.

---

## 1. 다루는 범위

- **이메일 + 패스워드 기반 신규 사용자 생성**
- 약관 동의 history 저장
- 가입 후 **이메일 인증 단계로 진입** (`PENDING_VERIFICATION` 상태)
- 도메인 이벤트 (`UserRegistered`) 발행 → 외부 부수효과 (인증 메일) 는 별도 listener

## 2. 다루지 않는 범위

| 영역 | 어디서 | 비고 |
| --- | --- | --- |
| 소셜 로그인 가입 (Apple / Google / Kakao / Naver) | [[../oauth2-social-login]] | provider 토큰 검증 + 자동 ACTIVE |
| 이메일 인증 토큰 발급 / verify endpoint | [[../email-verification]] | `PENDING_VERIFICATION → ACTIVE` 전이 |
| 패스워드 분실 / 재설정 | [[../password-reset]] | 단일 사용 토큰 |
| 로그인 (JWT 발급) | [[../login-jwt]] | 별도 |
| 2FA 등록 | [[../two-factor-auth]] | 가입 후 별도 |
| 전화번호 인증 (AlimTalk) | (미작성) | 한국 SaaS 의 SMS 인증 |
| 관리자 페이지에서의 사용자 직접 생성 | (admin/users) | 일반 가입 흐름과 분리 |

## 3. 사용 기술 스택

| 영역 | 선택 |
| --- | --- |
| Java | 21 LTS |
| Spring Boot | 3.3.x |
| Bean Validation | jakarta.validation |
| Password hash | argon2id (password4j) |
| DB | PostgreSQL 16 |
| ORM | JPA 기본 — MyBatis / 공존 변형은 [[../../database/jpa-mybatis-coexist#9.5.1]] |
| 마이그레이션 | Flyway |
| ID | ULID (app-side 발급) |
| 이메일 발송 | outbox + 별도 워커 (SMTP / SES) |
| 응답 envelope | `CommonResponse<T>` — [[../../common/response-envelope]] |
| 예외 처리 | `BusinessException` + `ApiExceptionHandler` |

## 4. 외부 의존성

- PostgreSQL 클러스터 (master 1 + replica 0~N)
- (옵션) Redis — 1시간 가입 rate limit
- (옵션) WAF / CAPTCHA — 봇 가입 방어
- (옵션) HIBP API (haveibeenpwned k-anonymity) — 유출 패스워드 차단

## 5. 이 레시피를 적용하면 좋은 상황

- 일반 B2C 서비스 / 이커머스 / SaaS — 이메일 + 패스워드 가입이 standard
- 가입 직후 이메일 인증을 요구하는 정책
- 약관 / 마케팅 동의의 **버전별 history** 가 필요한 정책 (GDPR / 한국 개정 정책)
- 도메인 이벤트 기반 비동기 fan-out 가능한 인프라 (Spring `ApplicationEventPublisher` / Kafka)

## 6. 과한 적용일 수 있는 상황

| 시나리오 | 더 간단한 대안 |
| --- | --- |
| 단순 내부 admin tool (사용자 5명) | DB 에 직접 INSERT + 간단 BCrypt — Aggregate / 도메인 이벤트 X |
| 어드민이 직접 사용자 추가만 — 가입 endpoint 없음 | `POST /admin/users` 단순 CRUD |
| SSO / 사내 LDAP / Keycloak 위임 | Spring Security OAuth2 client — 직접 가입 X |
| Anonymous 사용 (가입 전 임시 user) — 카트 등 | guest 모드 + 나중에 binding (signup 시점에 carry over) |
| `auth` 가 별도 서비스 (MSA 의 IDP) | IDP 가 가입 책임 — 본 서비스는 외부 user 받음 |

## 7. 적용 후 얻는 것 / 비용

**얻는 것**:
- 도메인 이벤트 기반 확장성 (이메일 인증 / 환영 메일 / BI 적재 / CRM 동기화)
- DB UNIQUE + Application 1차 검증 의 이중 안전망
- 평문 패스워드 0 보장 (Value Object 가 hash 만)
- 약관 동의의 audit trail
- 도메인 ↔ JPA 분리로 ORM 모드 변경 용이

**비용**:
- 파일 수 ↑ (User / UserId / Email / PasswordHash / UserStatus / UserRepository / UserJpaEntity / Adapter / DTO / UseCase / Controller / Exception handler / Listener — 약 13 클래스)
- 단순 CRUD 대비 코드량 2배
- 신규 팀원 onboarding 1~2일

→ **사용자 가입이 비즈니스 critical 한 일반 SaaS** 면 비용 대비 가치. **5명짜리 internal tool** 이면 과함.

## 8. 적용 전 결정해둘 것 (PR 전 체크)

- [ ] 이메일 + 패스워드 만 받나? (이름 / 전화 / 생년월일 추가?)
- [ ] 가입 직후 ACTIVE 인가, `PENDING_VERIFICATION` 인가? — 본 레시피는 후자
- [ ] 약관 — 필수 / 선택 분류 및 버전 관리 필요?
- [ ] enumeration 정책 — "이미 가입된 이메일" 직접 노출 (409) vs 항상 200 + 메일 안내?
- [ ] rate limiting — IP / email 별 정책
- [ ] `Idempotency-Key` 헤더 강제 여부

## 9. 관련

- [[signup|↑ signup hub]]
- [[requirements]] — 다음 (§1)
