---
title: "Java null safety / NullPointerException — Java 가 null 을 모르는 문제"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:30:00+09:00
tags:
  - backend
  - java-spring
  - pitfalls
  - null
  - npe
---

# Java null safety / NullPointerException — Java 가 null 을 모르는 문제

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 진단·전략 5가지 + record / Optional / @Nullable / Kotlin 비교 |

**[[pitfalls|↑ pitfalls hub]]**

---

## 1. 무엇이 문제

`null` 은 모든 reference 타입의 합법적 값. 컴파일러가 못 막음.

```java
String name = repo.findNameById(id);    // null 일 수 있는데 컴파일러 모름
name.length();                          // NPE
```

Tony Hoare 가 "null 은 내가 만든 the billion-dollar mistake" 라 한 이유.

### 1.1 NPE 가 자주 발생하는 자리

| 위치 | 패턴 |
| --- | --- |
| DB 컬럼 | 마이그레이션 잊은 nullable, 외래 키 의 자식 |
| 외부 API 응답 | JSON 의 optional 필드 |
| Map.get | 키 없으면 null |
| Array / List.get | OOB / sparse |
| Stream / Optional 연쇄 | `.get()` 무지성 |
| 의존성 주입 | `@Autowired` 빈 누락 (드물지만) |
| Constructor / Setter 누락 | partial 초기화 |
| Inheritance | 부모 setter 와 자식 getter 의 시점 차이 |

---

## 2. Java 17+ 의 무기 — Helpful NPE

Java 14+ 부터 NPE 메시지가 **어느 expression 이 null 인지** 알려줌.

```
Cannot invoke "String.length()" because the return value of
"com.example.UserRepository.findNameById(String)" is null
```

활성:
```bash
java -XX:+ShowCodeDetailsInExceptionMessages   # Java 14+
# Java 17+ 는 기본 ON
```

운영 로그 확인 첫 진단 도구.

---

## 3. null 회피 전략 — 5가지 (조합 사용)

### 전략 1 — `Optional<T>` (반환만)

```java
public Optional<User> findById(UserId id);

// 사용
users.findById(id)
    .map(User::email)
    .map(Email::value)
    .ifPresent(System.out::println);
```

**규칙**:
- ✅ 메서드 반환 타입
- ❌ 파라미터 (대신 overload)
- ❌ 필드 / 컬렉션 원소 / Map 값

```java
// ❌ 안티
class Order {
    private Optional<Address> shippingAddress;     // 필드에 Optional X
}

// ✅ 그냥 nullable + 보호
class Order {
    private Address shippingAddress;     // 또는 NullObject
    public Optional<Address> shippingAddress() {
        return Optional.ofNullable(shippingAddress);
    }
}
```

### 전략 2 — `record` + compact constructor

생성 시점에 검증:

```java
public record Email(String value) {
    public Email {
        Objects.requireNonNull(value, "email is required");
        if (value.isBlank()) throw new IllegalArgumentException("blank email");
    }
}
```

→ `Email` 가 일단 생성되면 `.value()` 는 절대 null 아님. **타입 시스템이 null 을 못 들어오게**.

### 전략 3 — `@Nullable` / `@NonNull` 어노테이션

```java
// org.jspecify (2024+ 표준), 또는 javax.annotation, JetBrains, Spring 의 @Nullable
import org.jspecify.annotations.NullMarked;
import org.jspecify.annotations.Nullable;

@NullMarked          // 패키지 / 모듈 default = NonNull
package com.example.shop.domain.user;

public class UserService {
    @Nullable
    public User findByEmail(Email email) { ... }       // null 가능

    public User getByEmail(Email email) { ... }        // null 절대 아님
}
```

→ IDE (IntelliJ) 가 컴파일 단에서 경고. 정적 분석 (NullAway, Checker Framework) 가능.

### 전략 4 — Null Object pattern

`null` 대신 "비어 있음" 객체:

```java
class User {
    private static final Address EMPTY_ADDRESS = new Address("", "", "", "");
    private Address shipping = EMPTY_ADDRESS;
    public Address shipping() { return shipping; }     // null 아님
}
```

→ caller 가 null 검사 안 해도 됨. 단 "비어 있음" 의 의미를 도메인에서 명시.

### 전략 5 — Defensive at boundary

도메인 안쪽은 null 없다고 가정. **boundary** (Controller / Repository / 외부 API) 에서만 변환.

```java
// Controller — Request 가 null 일 수 있음
public ApiResponse<...> create(@Valid @RequestBody @NotNull CreateProductRequest req) {
    // Bean Validation 이 null 막음
}

// Repository — DB null 컬럼 → Optional 또는 도메인 변환 시 검증
public Optional<User> findByEmail(Email email) { ... }

// 외부 API client — null 응답 정규화
public OrderInfo fetchOrder(String id) {
    var raw = restClient.get(...);
    return new OrderInfo(
        requireNonNull(raw.id(), "external API returned null id"),
        Optional.ofNullable(raw.note()).orElse(""),
        raw.amount()
    );
}
```

---

## 4. 실전 — Spring 에서

### 4.1 DB nullable 컬럼

```sql
-- 마이그레이션 V20__add_optional_phone.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);   -- nullable
```

```java
@Entity
class UserJpaEntity {
    @Column(nullable = true, length = 20)
    private String phone;                              // null 가능
}

// 도메인
public final class User {
    private final String phone;                        // null OK
    public Optional<String> phone() { return Optional.ofNullable(phone); }
}
```

> **함정**: `@Column(nullable=false)` 만 두고 마이그레이션 안 함 → DB 는 nullable 그대로. **DB 가 진실의 원천**. Flyway 검증 + JPA 일치.

### 4.2 DTO ↔ record + Bean Validation

```java
public record CreateOrderRequest(
    @NotNull String productId,                  // null 거부 → 422
    @NotBlank String shippingAddress,           // null + blank 거부
    @Positive int quantity,
    @Nullable String memo                       // null 허용 (기본 nullable, 명시 가능)
) {}
```

### 4.3 외부 JSON — Jackson 의 null

```java
// 기본 동작 — JSON 에 필드 없으면 null
{"id": "x"}                  → record.note() == null

// 안전 — default 또는 Optional
public record OrderInfo(
    String id,
    Optional<String> note          // ⚠️ 권장 X (필드에 Optional)
) {}

// 더 나은 방법 — 정규화 후 record
public record OrderInfo(String id, String note) {
    public OrderInfo {
        note = (note == null) ? "" : note;     // record compact constructor 에서 정규화
    }
}
```

### 4.4 Stream / Optional 연쇄

```java
// ❌ 안티
user.getProfile().getAddress().getCity();              // 어디서든 NPE

// ✅
Optional.ofNullable(user)
    .map(User::profile)
    .map(Profile::address)
    .map(Address::city)
    .orElse("UNKNOWN");

// ✅ 도메인에서 null 안 들고 다니기 (전략 4)
user.profile().address().city();                       // 모두 NonNull 보장이면 OK
```

### 4.5 컬렉션은 `null` 대신 `empty`

```java
// ❌
public List<Order> findByUserId(UserId id) {
    if (...) return null;
    return list;
}

// ✅
public List<Order> findByUserId(UserId id) {
    if (...) return List.of();
    return list;
}
```

caller 가 `null` 검사 없이 `for (var o : orders)` 가능.

### 4.6 `Map.get` 의 함정

```java
var price = priceMap.get("KRW");
price.toString();                          // null 가능

// ✅
var price = priceMap.getOrDefault("KRW", Money.ZERO);

// 또는
priceMap.computeIfAbsent("KRW", k -> Money.ZERO);
```

---

## 5. Optional 사용 함정

### Optional 함정 1 — `.get()` 무지성
```java
opt.get();         // 없으면 NoSuchElementException — NPE 와 동급 사고
```
**`.orElseThrow(...)`** 또는 `.orElse(...)` 사용.

### Optional 함정 2 — 파라미터 / 필드
```java
void process(Optional<User> user)        // ❌ — 호출자가 Optional.of(x) / Optional.empty() 갈팡질팡
void process(User user)                   // ✅ NonNull
void process()                           // empty 의미 → 메서드 분리 (overload)
```

### Optional 함정 3 — `isPresent() + get()` 패턴
```java
// ❌
if (opt.isPresent()) {
    var v = opt.get();
    ...
}

// ✅
opt.ifPresent(v -> { ... });
opt.map(v -> ...).orElse(...);
```

### Optional 함정 4 — 직렬화 (Jackson 의 Optional)
```java
public record OrderInfo(String id, Optional<String> note) {}
// JSON: {"id":"x", "note":{"present":false}}     ← 보기 끔찍
```
필드에 Optional 두지 말 것. 정규화 또는 raw `String` + `@Nullable`.

### Optional 함정 5 — 연쇄 vs 단순
```java
Optional.ofNullable(user)
    .map(User::profile)
    .map(Profile::address)
    .map(Address::city)
    .orElse("UNKNOWN");
```
도메인 모델이 깔끔하면 이렇게 길어질 일 자체가 없음. 길이가 길면 **도메인 모델 재고**.

---

## 6. Kotlin / Scala / TS 비교 (왜 다른 언어가 편한가)

| 언어 | null 표현 | 컴파일 검사 |
| --- | --- | --- |
| **Java** | 모든 reference 타입 | Helpful NPE / @Nullable annotation (선택) |
| **Kotlin** | `T?` (nullable type) | **컴파일러 강제** |
| **Scala 3** | `Option[T]` 표준 | 강함 |
| **TypeScript** | `T \| null \| undefined` | `strictNullChecks: true` 시 강함 |
| **Rust** | `Option<T>` | 컴파일러 강제 + 패턴 매칭 |

```kotlin
// Kotlin
val name: String? = repo.findNameById(id)
name.length                                  // ❌ 컴파일 에러
name?.length                                 // null 이면 null
name?.length ?: 0                           // 기본값
name!!.length                                // unsafe — null 이면 NPE
```

**Spring Boot 의 Kotlin 통합** — Spring 6 부터 `kotlin-spring` plugin + `@Nullable` annotation 자동 인식. Kotlin 으로 가면 NPE 가 거의 사라짐.

---

## 7. 정적 분석 도구

| 도구 | 특징 |
| --- | --- |
| **NullAway** (Uber, FB) | 가볍고 빠름. `@NullMarked` 와 잘 맞음 |
| **Checker Framework** | 엄격. Type system 수준 |
| **SpotBugs + FindBugs annotation** | 무난 |
| **IntelliJ IDEA inspection** | `@Nullable` / `@NonNull` 만으로 강력 |
| **Error Prone** (Google) | 컴파일 시 다양한 패턴 차단 |

```kotlin
// build.gradle.kts — NullAway 예
plugins { id("net.ltgt.errorprone") version "3.1.0" }
dependencies {
    errorprone("com.uber.nullaway:nullaway:0.10.20")
    errorprone("com.google.errorprone:error_prone_core:2.27.0")
}
tasks.withType<JavaCompile>().configureEach {
    options.errorprone.option("NullAway:AnnotatedPackages", "com.example.shop")
}
```

---

## 8. 권장 정책 (이 프로젝트 기준)

```
1. 도메인 모델 — record + compact constructor 로 null 봉쇄
2. 메서드 반환 — null 가능하면 Optional<T>, 컬렉션은 List.of() / Set.of()
3. 파라미터 — null 거부. Bean Validation (@NotNull) 또는 Objects.requireNonNull
4. 외부 boundary (Controller 입력, HTTP client 응답) — 정규화 후 도메인 진입
5. DB 컬럼 — nullable 명시 + JPA @Column(nullable=...) 일치
6. JSpecify @NullMarked + IntelliJ inspection 운영
7. Kotlin 으로 가면 7/10 사고 자동 차단
```

---

## 9. 함정 모음

### 함정 1 — `Objects.requireNonNull` 만 믿음
런타임 검사. 컴파일러는 못 막음. 보조 수단이지 최종 해법 아님.

### 함정 2 — Lombok `@NonNull` 의 의미
런타임 NPE 던지는 코드 생성. annotation 이지만 정적 검사는 아님.

### 함정 3 — `Optional` 을 자바빈 setter 에
`setX(Optional<String> x)` — 안 됨. Optional 은 반환만.

### 함정 4 — equals 에서 null 미고려
```java
return this.value.equals(other.value);    // value null 이면 NPE
return Objects.equals(this.value, other.value);   // 안전
```

### 함정 5 — `String.equals(literal)` vs `"literal".equals(s)`
```java
if (s.equals("admin")) ...                // s null → NPE
if ("admin".equals(s)) ...                // 안전
if (Objects.equals(s, "admin")) ...        // 명확
```

### 함정 6 — JPA `@Column(nullable=false)` 만 두고 마이그레이션 누락
JPA 어노테이션은 ddl-auto 가 아니면 DB 반영 안 됨. **DB schema 가 진실**.

### 함정 7 — Jackson 의 missing vs null
```json
{"a": null}     // a 는 명시적 null
{}              // a 는 missing
```
Java record 에선 둘 다 null. 구분이 필요하면 `JsonNullable<T>` (openapi-generator).

### 함정 8 — `Map.computeIfAbsent` 의 null value
값 자체가 null 이면 다시 계산. 의도와 다를 수 있음.

### 함정 9 — Stream 의 null 원소
`Stream.of(a, b, null)` 가능. `collect(toList())` 결과에 null 섞임. `filter(Objects::nonNull)`.

### 함정 10 — `@Async` 가 반환 `null`
Future 가 null 이면 caller `.get()` 에서 NPE. CompletableFuture 명시 + 예외 처리.

---

## 10. 운영 체크리스트

- [ ] Java 17+ + `-XX:+ShowCodeDetailsInExceptionMessages` (기본 on)
- [ ] 도메인 객체 `record` + compact constructor
- [ ] 메서드 반환에 `Optional<T>` 일관성
- [ ] 컬렉션 반환은 `List.of()` (null 금지)
- [ ] DTO 에 `@NotNull` / `@NotBlank` / `@Valid`
- [ ] JSpecify 또는 동등 annotation + IDE inspection
- [ ] NullAway 또는 Checker Framework (큰 프로젝트)
- [ ] 외부 API client 응답 정규화 단계 명시
- [ ] DB schema + JPA `nullable` 일치 확인 (Flyway test)
- [ ] 새 NPE 가 운영에 발생하면 root cause 분석 + 어디서 들어왔는지 boundary 찾기

---

## 11. 관련

- [[pitfalls|↑ pitfalls hub]]
- [[../api-design/signup]] — record + compact constructor 예
- [[../api-design/product-crud]] — 도메인 모델의 null 봉쇄
- [[transaction-pitfalls]] — Optional + JPA 의 self-invocation 함정
