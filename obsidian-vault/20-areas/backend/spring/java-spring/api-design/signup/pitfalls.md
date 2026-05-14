---
title: "signup §11 — 흔한 함정"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:54:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - pitfalls
---

# signup §11 — 흔한 함정

**[[signup|↑ signup hub]]**  ·  ← [[implementation-order]]

---

> 회원가입 구현 시 자주 발생하는 실수와 안티패턴. 코드 리뷰 시 체크리스트.

---

## 1. DB / 정합성

### 함정 1 — `email` 만 unique, `lower(email)` 안 함
`alice@x.com` 과 `Alice@X.com` 둘 다 통과 → 같은 사용자 2번 가입 가능.

**해결**: `CREATE UNIQUE INDEX ux_users_email ON users (lower(email));`

### 함정 2 — 1차 application 검증만, DB unique 누락
`existsByEmail` 통과한 두 트랜잭션이 동시에 INSERT → 둘 다 성공 (race).

**해결**: DB UNIQUE constraint 필수. `DataIntegrityViolationException` 잡아 409 매핑.

### 함정 3 — JPA `ddl-auto: update` 운영 사용
스키마 변경이 무언으로 일어남. 예측 불가능한 운영 사고.

**해결**: `ddl-auto: validate` + Flyway 가 진실의 원천.

### 함정 4 — DELETED user 가 같은 email 로 재가입 못 함
soft delete 후 같은 이메일 가입 시도 = UNIQUE 위반.

**해결**: DELETE 시 email anonymize (`deleted-{id}@deleted.example.com`) 또는 partial unique index (`WHERE status <> 'DELETED'`).

### 함정 5 — `password_hash VARCHAR(60)` (옛 BCrypt 길이)
argon2id PHC 가 ~120 자 → truncate → 로그인 실패.

**해결**: `VARCHAR(255)` 안전.

---

## 2. 보안 / 비밀번호

### 함정 6 — SHA-256 / MD5 로 password 해시
초당 수십억 시도 가능. 사실상 평문 저장과 동급.

**해결**: argon2id / bcrypt (cost ≥ 12) / scrypt 만.

### 함정 7 — 평문 password 가 로그에 노출
`SignupRequest` 의 기본 `toString()` 이 모든 필드 출력. logger.info(request) 시 평문 노출.

**해결**: `toString()` 오버라이드 — password 필드 `***` 으로 마스킹.

```java
public record SignupRequest(...) {
    @Override public String toString() {
        return "SignupRequest[email=%s, password=***, name=%s]"
            .formatted(email, name);
    }
}
```

### 함정 8 — Hibernate SQL 로그가 password_hash 노출
`logging.level.org.hibernate.SQL=DEBUG` 면 INSERT 의 password_hash 가 평문 (해시지만 노출).

**해결**: 운영 INFO 이상.

### 함정 9 — bcrypt 의 72-byte 컷
긴 passphrase 가 silently truncated. 사용자가 `"이게-내-매우-긴-비밀번호-입니다-72바이트-이상"` 입력하면 잘림.

**해결**: argon2id 사용 또는 SHA-256 pre-hash 후 bcrypt (Dropbox 방식).

### 함정 10 — Bean Validation 만으로 약한 password 차단 시도
`@Size(min=8)` 만으론 `"12345678"` 통과. NIST 약한 패스워드 차단 필요.

**해결**: `PasswordPolicy` 추가 (haveibeenpwned 통합).

### 함정 11 — 응답에 password / password_hash 노출
`SignupResponse` 에 모든 user 필드 포함 시 password_hash 도 응답 JSON 에.

**해결**: DTO 명시적 작성 (필요한 필드만).

---

## 3. 트랜잭션 / 동시성

### 함정 12 — `@Transactional` 안에서 SMTP / PG / 외부 API 호출
외부 IO 가 느리면 DB 락 시간 ↑ → connection pool 고갈.

**해결**: `@TransactionalEventListener(phase = AFTER_COMMIT)` + outbox 패턴.

### 함정 13 — Listener phase 가 `BEFORE_COMMIT` 인데 외부 호출
트랜잭션 rollback 시 — 메일은 발송됐는데 사용자는 가입 안 된 상태. 사용자 항의.

**해결**: 외부 부수효과는 `AFTER_COMMIT` 만.

### 함정 14 — `@Transactional` self-invocation
같은 클래스 내부 메서드 호출 = AOP proxy 미작동 → 트랜잭션 적용 X.

```java
class SignupService {
    public void a() { b(); }                          // ⚠️
    @Transactional public void b() { ... }            // 적용 X
}
```

**해결**: 별도 빈 분리 또는 self-injection.

### 함정 15 — 트랜잭션 격리 수준 오해
SERIALIZABLE 강제 = 과한 락 + 충돌 시 retry 필요. 일반적으로 READ COMMITTED 면 충분.

**해결**: 명시적으로 변경 X — 기본값 사용.

---

## 4. 도메인 모델

### 함정 16 — 도메인이 `@Entity` 와 결합
`User` 클래스에 `@Entity @Table` 어노테이션 → ORM 변경 시 도메인 재작성. lazy proxy 누수.

**해결**: 도메인 ↔ JPA Entity 분리. Adapter 가 매핑.

### 함정 17 — Setter 노출
`user.setStatus(...)` 가 public — 어디서든 status 변경 가능. 상태 머신 무의미.

**해결**: setter X. `verifyEmail() / suspend() / delete()` 같은 의미 있는 메서드만.

### 함정 18 — Aggregate 가 너무 큼
`User` 가 profile / orders / posts / followers 다 들고 있음 → 단일 transaction 에 너무 많은 entity load.

**해결**: Aggregate 경계 좁게. 다른 도메인은 ID 참조 + 별도 Aggregate.

### 함정 19 — Domain Event 없음
가입 후 이메일 / 알림 / BI 모두 SignupUseCase 가 직접 호출 → UseCase 가 외부 시스템 다 알아야 됨.

**해결**: `UserRegistered` 이벤트 + listener 들이 후속 처리.

### 함정 20 — Value Object 검증 누락
`Email("not-an-email")` 이 통과 → DB 에 garbage 저장.

**해결**: record compact constructor 에서 검증.

---

## 5. 검증 / 입력

### 함정 21 — Bean Validation 없이 컨트롤러 진입
`@Valid` 누락 → null body / 빈 필드가 service 까지 흘러감.

**해결**: 모든 `@RequestBody` 에 `@Valid`.

### 함정 22 — `@AssertTrue` 의 boolean 기본 false
`termsAgreed` 가 request 에 없으면 false → assertTrue 위반 → 422. 의도와 일치하지만 메시지 명확화 필요.

**해결**: `@AssertTrue(message = "must agree to terms")` 메시지 명시.

### 함정 23 — email 의 trim / lowercase 누락
`"  Alice@X.COM  "` 그대로 저장 → DB 의 다른 row 와 다르다고 인식.

**해결**: Application layer 에서 `email.trim().toLowerCase(Locale.ROOT)` 정규화.

### 함정 24 — 한국어 / 특수문자 name 거절
정규식이 영문만 허용 → 한국 사용자 가입 불가.

**해결**: `@Size` 만, regex 검증 X. Bean Validation 의 `@Pattern` 사용 시 unicode aware.

---

## 6. UX / 정책

### 함정 25 — Enumeration 무방비
"이미 가입된 이메일입니다" 직접 노출 = 가입자 목록 enumeration.

**해결 (보안 민감)**: 항상 200 + "이메일을 확인해 주세요" 동일 메시지. 실제 가입 / 안 가입은 backend 분기.

본 레시피 정책: 일반 SaaS 는 직관적 (409), 금융 / 보건은 enumeration 차단.

### 함정 26 — 가입 직후 ACTIVE
이메일 인증 없이 ACTIVE = 가짜 이메일 가입 가능. 봇 / 스팸.

**해결**: `PENDING_VERIFICATION → ACTIVE` 흐름 강제 ([[../email-verification]]).

### 함정 27 — 가입 직후 자동 로그인 (JWT 발급)
응답에 JWT 포함 = 가입과 로그인이 결합. 이메일 인증 흐름과 충돌.

**해결**: 가입은 user 생성만. 로그인은 별도 API.

### 함정 28 — 약관을 `users.marketing_agreed BOOLEAN` 으로
약관 버전 변경 / 재동의 / 철회 history 추적 불가. GDPR / 한국 개정 정책 위반.

**해결**: `user_terms_consent_history` 별도 테이블 + `terms` 버전 관리.

---

## 7. 운영

### 함정 29 — Rate limit 없음
한 IP 가 분당 1000 가입 시도 = 봇 의심 + 인프라 부담. argon2 CPU 부하 폭증.

**해결**: Bucket4j + Redis ([[../rate-limiting]]) — IP 별 1시간 3회.

### 함정 30 — JwtAuthenticationEntryPoint 의 401 자리에 400 응답 (job-answer-be 의 실제 버그)
다른 곳의 함정이지만 signup 흐름에 영향 — Security filter 통과 못 한 요청이 400 으로 와서 클라가 401 분기 X.

**해결**: `SC_UNAUTHORIZED` (401) 명시 ([[../../common/security-config#4.1]]).

### 함정 31 — Idempotency-Key 무시
네트워크 retry 로 같은 이메일 2번 시도 = 409 응답 → 클라 혼동 (방금 성공한 거 아닌가?).

**해결**: `Idempotency-Key` header optional 지원. 24시간 같은 응답.

### 함정 32 — outbox 워커 다운 미감지
인증 메일 안 나감 → 사용자 항의 → 운영 발견.

**해결**: `email_outbox.pending` Prometheus gauge + > 1000 알람.

---

## 8. 회귀 방지 — 코드 리뷰 체크리스트

PR 마다 확인:

- [ ] DB UNIQUE 인덱스 변경 시 정합성 검토
- [ ] DTO 가 password 필드 가지면 toString 마스킹
- [ ] 새 외부 호출이 트랜잭션 안에 있지 않은지
- [ ] 새 endpoint 가 SecurityConfig 정책 반영
- [ ] 새 도메인 이벤트는 listener / 메트릭 반영
- [ ] 시나리오 표 ([[testing#1]]) 갱신
- [ ] Bean Validation 누락 X
- [ ] Hibernate SQL 로그 level WARN 이상

---

## 9. 관련

- [[signup|↑ signup hub]]
- [[implementation-order]] — 이전 (§10)
- [[../../pitfalls/n-plus-one]] · [[../../pitfalls/null-safety]] — 다른 함정 영역
- [[../../common/response-envelope]] · [[../../common/security-config]] — 표준 패턴 (위반 시 함정)
