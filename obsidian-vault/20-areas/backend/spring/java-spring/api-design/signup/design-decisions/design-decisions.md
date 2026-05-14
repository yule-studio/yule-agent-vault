---
title: "auth §4 — 설계 의사결정 (Hub)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T04:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - design-decisions
  - hub
---

# auth §4 — 설계 의사결정 (Hub)

**[[../signup|↑ signup hub]]**  ·  ← [[../requirements]]  ·  → [[../database/database]]

> ⭐ **이 폴더의 핵심 결정 가이드.** 어떤 도구 / 알고리즘 / 정책을 골라야 하나 — 각 항목마다 별도 노트.
> 각 결정은 단순 권장 X — **왜 필요한가 / 안 하면 무슨 문제 / 대안과 왜 안 됨 / 트레이드오프** 4구조로.

---

## 1. 13 개 결정의 인덱스

### 1.1 인증 / 가입

| 노트 | 결정 | 본 vault 기본 |
| --- | --- | --- |
| [[auth-channels]] | 가입 인증 채널 | 이메일 + 휴대폰 (한국 SaaS) |
| [[password-hash]] | 패스워드 해시 알고리즘 | Argon2id (m=64MB, t=3, p=4) |
| [[token-model]] | JWT vs Session | JWT (HS256) + opaque refresh |
| [[refresh-storage]] | refresh 토큰 저장소 | RDB → Redis 마이그레이션 |

### 1.2 외부 통신

| 노트 | 결정 | 본 vault 기본 |
| --- | --- | --- |
| [[email-provider]] | 이메일 발송 도구 | AWS SES (서울 리전) |
| [[sms-provider]] | SMS / 알림톡 / 본인인증 | NCP SENS + AlimTalk + (필요시) PASS/NICE |
| [[social-login-providers]] | 소셜 로그인 | Kakao + Naver + Google + Apple |

### 1.3 보안 / 정책

| 노트 | 결정 | 본 vault 기본 |
| --- | --- | --- |
| [[two-factor-auth-policy]] | 2FA 도입 시점 | 옵션 + step-up + ADMIN 강제 |
| [[terms-consent-policy]] | 약관 / 개인정보 동의 | 별도 테이블 + 버전 관리 + 재동의 |
| [[enumeration-policy]] | 사용자 enumeration 방지 | 일반 SaaS = 직관적, 금융 = 차단 |
| [[idempotency-policy]] | Idempotency-Key | Optional (결제 / 주문은 Required) |

### 1.4 운영

| 노트 | 결정 | 본 vault 기본 |
| --- | --- | --- |
| [[dependencies]] | build.gradle 추천 stack | Spring Boot 3.3 + Java 21 + password4j + jjwt |
| [[default-stack]] | 의사결정 종합 결과 (yaml) | 본 vault 의 base config |

---

## 2. 결정의 순서 — 어떤 것부터

```
1. auth-channels      → 어떤 채널로 가입받나
2. password-hash      → 비밀번호 어떻게 저장하나
3. token-model        → 인증 토큰 무엇으로
4. refresh-storage    → refresh 어디에
5. email/sms-provider → 외부 통신 도구
6. social-login       → IdP 결정
7. terms-consent      → 법적 요구
8. 2FA / enumeration / idempotency → 보안 / 정책
9. dependencies       → 코드 의존성
```

**왜 이 순서**
- 위 결정이 아래 결정의 전제. password-hash 안 정하면 user 테이블 스키마 결정 X.
- token-model → refresh-storage → social-login 흐름은 의존성 있음.

---

## 3. 본 vault 의 표준 가정

본 vault 의 모든 결정은 **다음 컨텍스트** 를 가정:

| 항목 | 가정 |
| --- | --- |
| 서비스 종류 | 한국 B2C SaaS (커머스 / 미디어 / 소셜) |
| 트래픽 | MAU 10만 ~ 1000만 |
| 인프라 | AWS 서울 + RDS PostgreSQL + Redis |
| 인증 우선순위 | UX > 보안 (강한 보안은 금융 / 의료) |
| 글로벌 | 한국 우선, 글로벌 확장 가능 |

**다른 컨텍스트면 결정이 달라짐**
- B2B SaaS → 이메일 우선, SMS 무관.
- 글로벌 → AlimTalk X, Twilio.
- 금융 / 의료 → 본인인증 (PASS/NICE) 필수, enumeration 차단.

각 결정 노트의 "다른 컨텍스트" 섹션 참고.

---

## 4. 관련

- [[../signup|↑ signup hub]]
- [[../requirements]] — 이전 (§3)
- [[../database/database]] — 다음 (§5)
- [[../security]] — 보안 정책 (보안 강도의 깊이)
- [[../implementation-order]] — 어떤 순서로 구현할지
