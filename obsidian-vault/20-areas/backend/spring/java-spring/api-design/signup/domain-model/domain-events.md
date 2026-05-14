---
title: "Domain Events — 5종 + listener 패턴"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:15:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - domain-event
---

# Domain Events — 5종 + listener 패턴

**[[domain-model|↑ domain-model hub]]**

> 도메인의 상태 변경을 외부에 알리는 메커니즘. Aggregate 가 발행, listener 가 후속 처리.

---

## 1. 정의 — sealed interface

```java
// src/main/java/com/example/shop/domain/common/DomainEvent.java
public sealed interface DomainEvent
    permits UserRegistered, UserEmailVerified, UserPhoneVerified,
            UserPasswordChanged, UserSuspended, UserUnsuspended,
            UserDeleted, UserRoleChanged {
    Instant occurredAt();
}
```

> 💡 **왜 sealed?**
> Java 17+ — exhaustive switch 가능 (모든 이벤트 처리 강제).
> 새 event 추가 시 컴파일러가 switch 갱신 강제.

> 💡 **왜 `occurredAt`?**
> 모든 이벤트는 시간 정보 필요. (audit log / sequencing / replay)

---

## 2. 이벤트 5종 (auth 기준)

```java
public record UserRegistered(
    UserId userId,
    Email email,
    SocialProviderType providerType,
    Instant occurredAt
) implements DomainEvent {}

public record UserEmailVerified(
    UserId userId,
    Email email,
    Instant occurredAt
) implements DomainEvent {}

public record UserPhoneVerified(
    UserId userId,
    PhoneNumber phone,
    Instant occurredAt
) implements DomainEvent {}

public record UserPasswordChanged(
    UserId userId,
    Instant occurredAt
) implements DomainEvent {}

public record UserSuspended(
    UserId userId,
    String reason,
    UserId actorId,                      // 누가 정지했는지 (관리자 ID)
    Instant occurredAt
) implements DomainEvent {}

public record UserRoleChanged(
    UserId userId,
    Role oldRole,
    Role newRole,
    UserId changedBy,
    Instant occurredAt
) implements DomainEvent {}

public record UserDeleted(
    UserId userId,
    Instant occurredAt
) implements DomainEvent {}
```

> 💡 **왜 record?**
> 이벤트는 본질적으로 immutable data. record 가 정확한 도구.

> 💡 **왜 `UserId` 등 도메인 타입 그대로 (String 아닌)?**
> Listener 가 도메인 객체 다룰 때 타입 안전. JSON 직렬화 시 `@JsonValue` 로 String 변환.

---

## 3. Aggregate 가 이벤트 발행 — `pullDomainEvents` 패턴

```java
public final class User {
    private final List<DomainEvent> events = new ArrayList<>();

    public static User register(...) {
        var u = new User(...);
        u.events.add(new UserRegistered(id, email, providerType, now));   // 발행
        return u;
    }

    public void verifyEmail(Instant now) {
        ...
        events.add(new UserEmailVerified(id, email, now));
    }

    public List<DomainEvent> pullDomainEvents() {
        var copy = List.copyOf(events);
        events.clear();
        return copy;
    }
}
```

> 💡 **왜 in-memory list + pull 패턴?**
>
> 대안 — Aggregate 가 직접 `ApplicationEventPublisher.publishEvent(...)` 호출?
> ❌ — 도메인 layer 가 Spring 의존하게 됨.
>
> **도메인은 events 모아두기만**, **Application layer 가 pull + publish**.

---

## 4. Application Service 의 publish

```java
@Service
@RequiredArgsConstructor
public class SignupUseCase {

    private final UserRepository users;
    private final ApplicationEventPublisher events;

    @Transactional
    public User handle(SignupCommand cmd) {
        var user = User.register(...);
        var saved = users.save(user);

        // 도메인 events 회수 + publish
        saved.pullDomainEvents().forEach(events::publishEvent);

        return saved;
    }
}
```

→ Spring 의 `ApplicationEventPublisher` 가 listener 들에 dispatch.

---

## 5. Listener — `@TransactionalEventListener(AFTER_COMMIT)`

```java
@Component
@RequiredArgsConstructor
public class UserRegisteredEmailListener {

    private final EmailOutboxRepository outbox;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(UserRegistered event) {
        outbox.enqueue(new EmailOutboxRow(
            event.email().value(),
            "verification",
            Map.of("userId", event.userId().value()),
            event.occurredAt()
        ));
    }
}
```

> 💡 **왜 `AFTER_COMMIT`?**
> 트랜잭션 rollback 가능성 — 만약 `BEFORE_COMMIT` 이면 — DB rollback 됐는데 메일은 발송 = 사고.
> **AFTER_COMMIT** = 커밋 성공 시점에만 listener 실행. 안전.

> 💡 **왜 `BEFORE_COMMIT` 도 사용?**
> outbox INSERT 처럼 **같은 트랜잭션 안에 있어야 일관성** 인 경우만.
> 외부 IO (SMTP / Kafka / API) 는 항상 `AFTER_COMMIT`.

### 5.1 phase 종류

| phase | 시점 | 사용 |
| --- | --- | --- |
| `BEFORE_COMMIT` | commit 직전 | 같은 트랜잭션의 추가 DB 작업 (outbox INSERT 등) |
| `AFTER_COMMIT` | commit 성공 후 | 외부 IO 전부 (이메일 / Kafka / API) |
| `AFTER_ROLLBACK` | rollback 후 | 보상 작업 / 알람 |
| `AFTER_COMPLETION` | commit / rollback 둘 다 후 | cleanup |

---

## 6. 비동기 listener — `@Async`

```java
@Component
@RequiredArgsConstructor
public class UserRegisteredAnalyticsListener {

    private final AnalyticsClient analytics;

    @Async                                    // 별도 스레드 풀
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(UserRegistered event) {
        analytics.track("signup", Map.of("userId", event.userId().value()));
    }
}
```

```java
@EnableAsync
@Configuration
public class AsyncConfig {
    @Bean(name = "applicationEventMulticaster")
    public ApplicationEventMulticaster multicaster() {
        var multicaster = new SimpleApplicationEventMulticaster();
        multicaster.setTaskExecutor(taskExecutor());
        return multicaster;
    }

    @Bean
    public Executor taskExecutor() {
        var executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("async-event-");
        executor.initialize();
        return executor;
    }
}
```

> 💡 **`@Async` listener 의 함정**:
> - 예외 처리 — 별도 thread → 호출자 try-catch 무용 — `AsyncUncaughtExceptionHandler` 필요
> - 트랜잭션 propagation — `@Async` 메서드는 caller 트랜잭션과 별개

---

## 7. 같은 event 의 여러 listener

```java
@Component
public class UserRegisteredEmailListener {
    @TransactionalEventListener(AFTER_COMMIT)
    public void on(UserRegistered event) { /* 이메일 발송 */ }
}

@Component
public class UserRegisteredCrmListener {
    @TransactionalEventListener(AFTER_COMMIT)
    public void on(UserRegistered event) { /* CRM 동기화 */ }
}

@Component
public class UserRegisteredAnalyticsListener {
    @TransactionalEventListener(AFTER_COMMIT)
    public void on(UserRegistered event) { /* BI 적재 */ }
}
```

→ Spring 이 모든 listener 호출. 순서는 `@Order` 로 제어 가능 (단 의존 없는 게 정석).

---

## 8. 이벤트 → Kafka 외부 발행

```java
@Component
@RequiredArgsConstructor
public class UserEventKafkaPublisher {

    private final KafkaTemplate<String, String> kafka;
    private final ObjectMapper json;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void on(UserRegistered event) {
        try {
            kafka.send("user-events",
                event.userId().value(),               // partition key
                json.writeValueAsString(event));
        } catch (Exception e) {
            log.warn("Kafka publish failed", e);
            // best-effort — 비즈니스 흐름 영향 X
        }
    }
}
```

→ 외부 시스템 (다른 service / CRM / BI) 이 Kafka consume.

**더 안전 — Transactional Outbox pattern**:
- 트랜잭션 안에서 `outbox_events` 테이블에 INSERT
- 별도 worker 가 outbox → Kafka 발행
- 보장: at-least-once delivery

자세히: [[../../webhook-send]] — 같은 패턴.

---

## 9. event 의 모양 결정

### 9.1 thin vs thick

```java
// Thin — ID 만
public record UserRegistered(UserId userId, Instant occurredAt) {}

// Thick — 데이터 다 포함
public record UserRegistered(
    UserId userId, Email email, String name,
    SocialProviderType providerType, Instant occurredAt
) {}
```

| | Thin | Thick |
| --- | --- | --- |
| listener 가 DB 조회 필요 | ✅ | ❌ |
| event payload 큼 | ❌ | ⚠️ |
| user 정보 변경 시 stale | OK (다시 조회) | stale 가능 |

본 vault: **Thick (필요한 정보만)**. listener 가 DB 추가 호출 안 해도 됨.

### 9.2 비밀 정보는 X

```java
public record UserRegistered(
    UserId userId, Email email,
    PasswordHash passwordHash         // ❌ 절대 X
) {}
```

→ event 는 listener / Kafka / log 에 전파 가능. 비밀은 event 에 안 넣음.

---

## 10. 함정 모음

### 함정 1 — Aggregate 가 직접 publish
도메인 layer 가 Spring 의존. **pull 패턴**.

### 함정 2 — `BEFORE_COMMIT` 에서 외부 호출
rollback 시 사고. **AFTER_COMMIT**.

### 함정 3 — listener 의 예외가 caller 영향
sync listener (`@TransactionalEventListener` 만) 의 예외 → caller 까지 전파. AFTER_COMMIT 이라도 — 비즈니스 흐름 깨질 수 있음.
**try-catch + log** 또는 `@Async`.

### 함정 4 — pullDomainEvents 두 번
같은 이벤트 두 번 publish. clear() 가 보장하지만, 둘 다 호출 안 함을 caller 가 책임.

### 함정 5 — event 에 사용자 input 그대로
검증 안 된 데이터가 event 에. 도메인이 검증한 값 (VO) 만 event 에.

### 함정 6 — sealed interface 안 사용
새 event 추가 시 모든 listener / switch 갱신 누락. **sealed**.

### 함정 7 — event 의 순서 의존
다른 listener 간 순서 의존하지 말 것 (테스트 불가). `@Order` 는 응급.

### 함정 8 — Kafka 동기 발행
KafkaTemplate.send().get() 으로 blocking → 트랜잭션 길어짐. async 또는 outbox.

### 함정 9 — event 에 mutable object
record 의 List 필드 — caller 가 변경 가능. `List.copyOf` 또는 컬렉션 자체 안 노출.

### 함정 10 — `@TransactionalEventListener` 이 트랜잭션 밖에서 호출
caller 에 트랜잭션 없으면 — listener 미실행 (warning). **`@Transactional` 안에서만 publish**.

---

## 11. 관련

- [[domain-model|↑ domain-model hub]]
- [[user-aggregate]] — event 발행 위치
- [[repository-ports]] — listener 가 Repository 호출 시
- [[../../webhook-send]] — 외부 시스템 발행 (outbox 패턴)
- [[../signup-impl#10 Infrastructure — UserRegisteredEmailListener]] — 적용 예
