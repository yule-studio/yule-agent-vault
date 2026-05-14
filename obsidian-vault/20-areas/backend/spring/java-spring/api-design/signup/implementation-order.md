---
title: "signup §10 — 구현 순서 (단계별 to-do)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:52:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - implementation-order
---

# signup §10 — 구현 순서 (단계별 to-do)

**[[signup|↑ signup hub]]**  ·  ← [[operations]]  ·  → [[pitfalls]]

---

> 이 레시피를 처음 만들 때 **어느 파일부터 만들지** 정해주는 단계별 가이드.
> 각 단계는 PR 1개로 나눌 수 있을 만큼 작게 — bottom-up (DB → domain → application → presentation) 권장.

---

## 1. 요구사항 / 완료 조건 작성 (Issue / PR 본문)

### To-do
- [ ] [[requirements#2 완료 조건]] 의 형식으로 **Given / When / Then** 작성
- [ ] API spec (request / response 예시) 결정
- [ ] 비기능 (latency / 동시성 / 멱등성) 정책 결정
- [ ] enumeration 정책 (409 vs 200) 결정
- [ ] Out of scope 명시

**Deliverable**: GitHub Issue + design doc (1~2 페이지)

---

## 2. DB 스키마 + Flyway 마이그레이션

### To-do
- [ ] [[database#2 스키마]] 의 `users` 테이블 DDL
- [ ] `lower(email)` UNIQUE 인덱스
- [ ] `user_terms_consent_history` + `terms` 테이블 (약관)
- [ ] `email_outbox` 테이블 (인증 메일)
- [ ] Flyway V1, V2, V3 파일 생성
- [ ] 로컬에서 마이그레이션 dry run

**Deliverable**: `src/main/resources/db/migration/V1__create_users.sql` 등

---

## 3. Domain Layer — Value Object + Aggregate

### To-do
- [ ] `domain/user/Email.java` (record + 검증)
- [ ] `domain/user/PasswordHash.java`
- [ ] `domain/user/UserId.java`
- [ ] `domain/user/UserStatus.java` (enum)
- [ ] `domain/common/DomainEvent.java` (sealed interface)
- [ ] `domain/user/events/UserRegistered.java`, `UserEmailVerified.java`, `UserPasswordChanged.java`
- [ ] `domain/user/User.java` (Aggregate Root + 상태 전이)
- [ ] **단위 테스트** — `UserTest`, `EmailTest`, `PasswordHashTest`

**Deliverable**: 도메인 layer + 단위 테스트 통과 (DB / Spring 의존 X)

---

## 4. Domain Layer — Repository Port (Interface)

### To-do
- [ ] `domain/user/UserRepository.java`
- [ ] `domain/user/PasswordEncoder.java`
- [ ] `domain/common/IdGenerator.java`
- [ ] `domain/user/exceptions/EmailAlreadyExistsException.java`

> **interface 만** — 아직 구현 X. 이게 도메인 ↔ infrastructure 의 contract.

---

## 5. Infrastructure — Persistence Adapter

### To-do
- [ ] `infrastructure/persistence/jpa/user/UserJpaEntity.java` (`@Entity`)
- [ ] `infrastructure/persistence/jpa/user/UserJpaRepository.java` (Spring Data interface)
- [ ] `infrastructure/persistence/jpa/user/JpaUserRepositoryAdapter.java` (implements UserRepository)
- [ ] 도메인 ↔ JPA 매핑 (`toDomain`, `apply`)
- [ ] **Testcontainers 통합 테스트** — `JpaUserRepositoryAdapterIT`
  - INSERT / findByEmail / existsByEmail / case-insensitive

**Deliverable**: DB 와 통신하는 first vertical slice

---

## 6. Infrastructure — 다른 Adapter

### To-do
- [ ] `infrastructure/security/Argon2PasswordEncoder.java` (implements PasswordEncoder)
  - password4j 라이브러리
  - 단위 테스트 — encode / matches
- [ ] `infrastructure/id/UlidIdGenerator.java`
- [ ] (옵션) `infrastructure/external/hibp/PwnedPasswordChecker.java`

---

## 7. Application — UseCase

### To-do
- [ ] `application/user/SignupCommand.java`
- [ ] `application/user/PasswordPolicy.java` (`@Component`)
- [ ] `application/user/SignupUseCase.java` (`@Service @Transactional`)
  - termsAgreed 검증
  - passwordPolicy.validate
  - email 정규화 (lowercase)
  - users.existsByEmail
  - User.register
  - users.save + DataIntegrityViolation 처리
  - termsConsent.save
  - events.publishEvent
- [ ] **단위 테스트** — `SignupUseCaseTest` (Mockito)

**Deliverable**: 비즈니스 로직 완성. DB / HTTP 무관하게 unit test 통과.

---

## 8. Presentation — Controller + DTO

### To-do
- [ ] `presentation/api/v1/auth/SignupRequest.java` (record + `toString` 마스킹)
- [ ] `presentation/api/v1/auth/SignupResponse.java`
- [ ] `presentation/api/v1/auth/SignupController.java`
  - `@Valid @RequestBody`
  - `@Operation` / `@ApiResponse` (Swagger)
  - `@SecurityRequirements()` (비인증)
  - DTO → Command 변환
- [ ] `@WebMvcTest` 로 Controller 단 검증 (Bean Validation)

---

## 9. 글로벌 예외 처리

### To-do
- [ ] `common/handler/ApiExceptionHandler.java` 가 이미 있다면 그대로 — [[../../common/response-envelope#6]]
- [ ] 없으면 — 표준 패턴 그대로 작성
- [ ] `EmailAlreadyExistsException` 이 `BusinessException` 상속하므로 자동 매핑 OK
- [ ] `DataIntegrityViolationException` 핸들러 별도 (안전망)

---

## 10. SecurityConfig

### To-do
- [ ] [[../../common/security-config]] 의 표준 그대로
- [ ] `/api/v1/auth/signup` 을 `permitAll` 에 추가
- [ ] CORS / CSRF / STATELESS 설정 확인
- [ ] (옵션) Rate limit filter 등록 — `RateLimitFilter`

---

## 11. Domain Event Listener — 이메일 outbox

### To-do
- [ ] `infrastructure/messaging/UserRegisteredEmailListener.java`
- [ ] `@TransactionalEventListener(phase = AFTER_COMMIT)`
- [ ] `EmailOutboxRepository.enqueue(...)`
- [ ] 통합 테스트 — 트랜잭션 커밋 후에만 outbox 적재 확인

---

## 12. 통합 테스트

### To-do
- [ ] `SignupApiIT` ([[testing#4]] 참고)
- [ ] [[testing#1 시나리오 표]] 의 🔴 必 시나리오 모두 통과
- [ ] CI 파이프라인에 등록

---

## 13. 운영 환경 준비

### To-do
- [ ] 환경변수 명세 ([[operations#1]])
- [ ] Prometheus 메트릭 export
- [ ] 로그 마스킹 검증
- [ ] Vault / Secrets Manager 의 secret 등록
- [ ] WAF / CAPTCHA (필요 시)

---

## 14. 코드 리뷰 체크리스트

PR 리뷰 시 확인:

### 도메인
- [ ] Value Object 의 검증이 생성자에서 (compact constructor)
- [ ] Aggregate 의 상태 변경은 메서드를 통해 (setter X)
- [ ] 도메인 이벤트가 모든 상태 변경에 발행

### Application
- [ ] `@Transactional` 이 메서드에 (Controller / Repository X)
- [ ] 외부 IO (SMTP / Kafka / HTTP) 가 트랜잭션 밖
- [ ] DataIntegrityViolation → 도메인 예외 변환

### Infrastructure
- [ ] JPA Entity 가 도메인 layer 외부에 (Adapter 안)
- [ ] `findByEmailIgnoreCase` 가 `lower(email)` 사용
- [ ] `@Version` 컬럼 (낙관 락)

### Presentation
- [ ] DTO `toString` 의 password 마스킹
- [ ] Bean Validation 어노테이션 명확
- [ ] Swagger `@Operation` / `@ApiResponse` 작성

### 테스트
- [ ] 시나리오 표의 🔴 必 모두 cover
- [ ] race condition 통합 테스트

### 보안
- [ ] 응답에 password 안 나감
- [ ] 로그에 password 안 나감
- [ ] argon2id 사용 (bcrypt 도 OK, 단 MD5/SHA 절대 X)

---

## 15. 전체 to-do 요약 (1 페이지)

```
[ ] 1.  Issue + 완료 조건 작성
[ ] 2.  Flyway V1/V2/V3 (users / terms / outbox)
[ ] 3.  Domain — Value Object + User Aggregate + 단위 테스트
[ ] 4.  Domain — Repository / PasswordEncoder / IdGenerator port (interface)
[ ] 5.  Infrastructure — JpaUserRepositoryAdapter + Testcontainers IT
[ ] 6.  Infrastructure — Argon2PasswordEncoder / UlidIdGenerator
[ ] 7.  Application — SignupUseCase / PasswordPolicy / 단위 테스트
[ ] 8.  Presentation — Controller / DTO + WebMvc 테스트
[ ] 9.  공통 — ApiExceptionHandler 확인 / 보강
[ ] 10. SecurityConfig — permitAll + Rate limit
[ ] 11. Listener — UserRegisteredEmailListener (AFTER_COMMIT)
[ ] 12. 통합 테스트 — SignupApiIT 의 13 시나리오
[ ] 13. 운영 준비 — env / 메트릭 / 로그 / WAF
[ ] 14. PR 리뷰
[ ] 15. 배포 후 smoke test
```

→ 1~13 까지 끝나면 **이 문서만 보고 처음부터 끝까지 구현 완료**.

---

## 16. 관련

- [[signup|↑ signup hub]]
- [[operations]] — 이전 (§9)
- [[pitfalls]] — 다음 (§11)
- [[testing]] — 시나리오 표
