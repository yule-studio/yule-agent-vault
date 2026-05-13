---
title: "MyBatis — 본편 (Spring Boot 3)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:00:00+09:00
tags:
  - backend
  - java-spring
  - database
  - mybatis
---

# MyBatis — 본편 (Spring Boot 3)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | mapper / dynamic SQL / typeHandler / 페이지네이션 / 함정 |

**[[database|↑ database hub]]**

> JPA 와 비교: [[jpa]] / 둘 다 사용 시: [[jpa-mybatis-coexist]].

---

## 1. 한 줄 정의

**MyBatis** = "SQL 을 직접 쓰고, 그 결과를 Java 객체에 매핑" 하는 프레임워크.
ORM 의 자동화 (변경 감지 / lazy / cascade) 가 없는 대신, **SQL 을 그대로 제어**.

한국 SI / 금융권 / 거대 레거시에서 표준. 신규 SaaS 는 JPA 기본 + MyBatis 보고서 패턴이 흔함.

---

## 2. 표준 의존성

```kotlin
// build.gradle.kts
implementation("org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3")
// PageHelper (페이지네이션)
implementation("com.github.pagehelper:pagehelper-spring-boot-starter:2.1.0")
```

```yaml
# application.yml
mybatis:
  mapper-locations: classpath:mapper/**/*.xml
  type-aliases-package: com.example.shop.domain
  configuration:
    map-underscore-to-camel-case: true        # user_name → userName
    default-statement-timeout: 5               # 초
    cache-enabled: false                       # 2차 캐시 (보통 끔, 외부 캐시 사용)
    jdbc-type-for-null: NULL
  type-handlers-package: com.example.shop.infrastructure.persistence.mybatis.typehandler
```

---

## 3. Mapper — XML 패턴

```
src/main/resources/mapper/user/UserMapper.xml
src/main/java/.../mybatis/UserMapper.java
```

```java
// UserMapper.java — 인터페이스
@Mapper
public interface UserMapper {
    Optional<UserRow> findById(@Param("id") String id);
    Optional<UserRow> findByEmail(@Param("email") String email);
    int insert(UserRow row);
    int updatePassword(@Param("id") String id, @Param("hash") String hash);
    int delete(@Param("id") String id);
    List<UserRow> search(UserSearchCriteria criteria);
}

// UserRow.java — DTO (또는 record)
public record UserRow(
    String id, String email, String name, String passwordHash,
    String status, Instant createdAt, Instant updatedAt
) {}
```

```xml
<!-- mapper/user/UserMapper.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.shop.infrastructure.persistence.mybatis.UserMapper">

    <resultMap id="UserRowMap" type="com.example.shop.infrastructure.persistence.mybatis.UserRow">
        <id     property="id"           column="id"/>
        <result property="email"        column="email"/>
        <result property="name"         column="name"/>
        <result property="passwordHash" column="password_hash"/>
        <result property="status"       column="status"/>
        <result property="createdAt"    column="created_at"/>
        <result property="updatedAt"    column="updated_at"/>
    </resultMap>

    <select id="findById" resultMap="UserRowMap">
        SELECT id, email, name, password_hash, status, created_at, updated_at
        FROM users
        WHERE id = #{id}
    </select>

    <select id="findByEmail" resultMap="UserRowMap">
        SELECT id, email, name, password_hash, status, created_at, updated_at
        FROM users
        WHERE lower(email) = lower(#{email})
    </select>

    <insert id="insert">
        INSERT INTO users (id, email, password_hash, name, status, created_at, updated_at)
        VALUES (#{id}, #{email}, #{passwordHash}, #{name}, #{status}, #{createdAt}, #{updatedAt})
    </insert>

    <update id="updatePassword">
        UPDATE users
        SET password_hash = #{hash}, updated_at = now()
        WHERE id = #{id}
    </update>

    <delete id="delete">
        DELETE FROM users WHERE id = #{id}
    </delete>

</mapper>
```

### 3.1 어노테이션 mapper (XML 안 쓰고)

```java
@Mapper
public interface UserMapper {

    @Select("SELECT id, email, name, password_hash AS passwordHash, status, " +
            "       created_at AS createdAt, updated_at AS updatedAt " +
            "FROM users WHERE lower(email) = lower(#{email})")
    Optional<UserRow> findByEmail(@Param("email") String email);

    @Insert("INSERT INTO users (id, email, password_hash, name, status, created_at, updated_at) " +
            "VALUES (#{id}, #{email}, #{passwordHash}, #{name}, #{status}, #{createdAt}, #{updatedAt})")
    int insert(UserRow row);
}
```

→ 단순 쿼리는 어노테이션, 동적 / 복잡 / 멀티라인은 XML. **한 프로젝트에서 하나 통일** 권장.

---

## 4. Dynamic SQL — XML 의 핵심 강점

```xml
<select id="search" resultMap="UserRowMap">
    SELECT id, email, name, status, created_at FROM users
    <where>
        <if test="email != null and email != ''">
            AND lower(email) LIKE concat('%', lower(#{email}), '%')
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
        <if test="statuses != null and statuses.size() > 0">
            AND status IN
            <foreach collection="statuses" item="s" open="(" separator="," close=")">
                #{s}
            </foreach>
        </if>
        <if test="createdFrom != null">
            AND created_at >= #{createdFrom}
        </if>
    </where>
    <choose>
        <when test="sort == 'LATEST'"> ORDER BY created_at DESC </when>
        <when test="sort == 'NAME'">   ORDER BY name ASC </when>
        <otherwise>                     ORDER BY id DESC </otherwise>
    </choose>
    LIMIT #{limit} OFFSET #{offset}
</select>
```

→ JPA Specification 보다 **SQL 가독성 높음**. 복잡 검색 / 보고서 SQL 의 최강.

### 4.1 `<bind>` — 가공 변수

```xml
<select id="searchByPrefix" resultMap="UserRowMap">
    <bind name="emailPattern" value="email + '%'"/>
    SELECT ... FROM users WHERE email LIKE #{emailPattern}
</select>
```

### 4.2 `<sql>` 재사용

```xml
<sql id="userColumns">
    id, email, name, password_hash AS passwordHash,
    status, created_at AS createdAt, updated_at AS updatedAt
</sql>

<select id="findById" resultMap="UserRowMap">
    SELECT <include refid="userColumns"/> FROM users WHERE id = #{id}
</select>
```

---

## 5. 결과 매핑 — resultMap / association / collection

### 5.1 1:1

```xml
<resultMap id="OrderDetailMap" type="OrderRow">
    <id     property="id"     column="o_id"/>
    <result property="amount" column="o_amount"/>
    <association property="buyer" javaType="UserRow">
        <id     property="id"    column="u_id"/>
        <result property="email" column="u_email"/>
    </association>
</resultMap>

<select id="findOrderWithBuyer" resultMap="OrderDetailMap">
    SELECT
        o.id AS o_id, o.total_amount_krw AS o_amount,
        u.id AS u_id, u.email AS u_email
    FROM orders o JOIN users u ON o.buyer_id = u.id
    WHERE o.id = #{id}
</select>
```

### 5.2 1:N

```xml
<resultMap id="ProductWithOptionsMap" type="ProductRow">
    <id     property="id"   column="p_id"/>
    <result property="name" column="p_name"/>
    <collection property="options" ofType="ProductOptionRow">
        <id     property="id"    column="po_id"/>
        <result property="name"  column="po_name"/>
        <result property="value" column="po_value"/>
        <result property="stock" column="po_stock"/>
    </collection>
</resultMap>

<select id="findProductWithOptions" resultMap="ProductWithOptionsMap">
    SELECT
        p.id AS p_id, p.name AS p_name,
        po.id AS po_id, po.name AS po_name, po.value AS po_value, po.stock AS po_stock
    FROM products p
    LEFT JOIN product_options po ON po.product_id = p.id
    WHERE p.id = #{id}
</select>
```

→ JPA 의 `@OneToMany` 와 비슷한 결과를 단일 쿼리로. cartesian product 주의.

---

## 6. 페이지네이션

### 6.1 직접 LIMIT / OFFSET (가장 명확)

```java
public record UserSearchCriteria(String email, String status, int limit, int offset) {}

@Select("SELECT ... FROM users WHERE ... LIMIT #{limit} OFFSET #{offset}")
List<UserRow> search(UserSearchCriteria criteria);

@Select("SELECT count(*) FROM users WHERE ...")
long count(UserSearchCriteria criteria);
```

### 6.2 PageHelper (라이브러리)

```java
@Mapper
public interface UserMapper {
    List<UserRow> findAll();
}

// 사용
PageHelper.startPage(pageNumber, pageSize);
var rows = userMapper.findAll();
var page = new PageInfo<>(rows);
// page.getTotal(), page.getPages(), page.getList()
```

→ SQL 안 건드리고 LIMIT 자동 추가. 단 **순서 의존적** (`startPage` 직후 첫 query 에만 적용). 위험.

권장: 명시적 LIMIT/OFFSET 또는 cursor (직접 작성).

---

## 7. 커서 (keyset) 페이지네이션

```xml
<select id="search" resultMap="UserRowMap">
    SELECT ... FROM users
    <where>
        AND status = 'ACTIVE'
        <if test="lastCreatedAt != null and lastId != null">
            AND (created_at, id) &lt; (#{lastCreatedAt}, #{lastId})
        </if>
    </where>
    ORDER BY created_at DESC, id DESC
    LIMIT #{limit}
</select>
```

→ [[../api-design/product-search]] §2 의 cursor 패턴을 SQL 직접.

---

## 8. typeHandler — 도메인 타입 ↔ DB

```java
// src/main/java/com/example/shop/infrastructure/persistence/mybatis/typehandler/EmailTypeHandler.java
@MappedTypes(Email.class)
@MappedJdbcTypes(JdbcType.VARCHAR)
public class EmailTypeHandler extends BaseTypeHandler<Email> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Email parameter, JdbcType jdbcType)
        throws SQLException { ps.setString(i, parameter.value()); }

    @Override
    public Email getNullableResult(ResultSet rs, String columnName) throws SQLException {
        var s = rs.getString(columnName); return s == null ? null : new Email(s);
    }
    @Override public Email getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        var s = rs.getString(columnIndex); return s == null ? null : new Email(s);
    }
    @Override public Email getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        var s = cs.getString(columnIndex); return s == null ? null : new Email(s);
    }
}
```

→ DB `varchar` ↔ `Email` Value Object 자동 변환.

### 8.1 enum

```java
// EnumTypeHandler 또는 mybatis 의 기본 EnumTypeHandler
@MappedTypes(OrderStatus.class)
public class OrderStatusTypeHandler extends EnumTypeHandler<OrderStatus> {
    public OrderStatusTypeHandler() { super(OrderStatus.class); }
}
```

> default `EnumTypeHandler` 가 enum 이름 (String) 으로 저장. 안전.

### 8.2 JSONB

```java
@MappedTypes(Map.class)
@MappedJdbcTypes(JdbcType.OTHER)         // PG 의 jsonb
public class JsonbMapTypeHandler extends BaseTypeHandler<Map<String, Object>> {
    private final ObjectMapper json = new ObjectMapper();

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Map<String, Object> v, JdbcType t) throws SQLException {
        try {
            var pgo = new PGobject();
            pgo.setType("jsonb");
            pgo.setValue(json.writeValueAsString(v));
            ps.setObject(i, pgo);
        } catch (JsonProcessingException e) { throw new SQLException(e); }
    }
    // get... 도 비슷
}
```

---

## 9. 트랜잭션

JPA 와 동일 — `@Transactional` 사용. Spring 의 트랜잭션 매니저가 MyBatis SqlSession 의 commit/rollback 도 처리.

```java
@Service
public class UserService {
    private final UserMapper users;

    @Transactional
    public void updateName(String id, String newName) {
        var row = users.findById(id).orElseThrow();
        users.updateName(id, newName);
    }
}
```

> **JPA 와 다른 점**: dirty checking 없음. **명시적으로 UPDATE 호출 필수**. 객체 setter 변경만으론 DB 반영 X.

---

## 10. Repository Adapter — 도메인 격리

JPA 와 같은 패턴 ([[../api-design/signup#6.5]] 참고):

```java
// 도메인 port
public interface UserRepository {
    Optional<User> findByEmail(Email email);
    User save(User user);
}

// MyBatis adapter
@Repository
public class MyBatisUserRepository implements UserRepository {

    private final UserMapper mapper;
    private final Clock clock;

    @Override
    public Optional<User> findByEmail(Email email) {
        return mapper.findByEmail(email.value()).map(this::toDomain);
    }

    @Override
    public User save(User user) {
        var existing = mapper.findById(user.id().value());
        if (existing.isEmpty()) {
            mapper.insert(toRow(user, Instant.now(clock)));
        } else {
            mapper.update(toRow(user, Instant.now(clock)));
        }
        return user;
    }

    private User toDomain(UserRow r) {
        return User.reconstitute(...);
    }
    private UserRow toRow(User u, Instant now) { ... }
}
```

→ Service 입장에서는 JPA / MyBatis 둘 다 같은 `UserRepository`.

---

## 10.5 Recipe 별 MyBatis 적용

[[../api-design/api-design#0.5 ORM 정책]] 에 따라, 각 레시피의 Repository port 를 MyBatis 로 구현할 때의 모범. **MyBatis only** 또는 **공존** 모드에서 사용.

### 10.5.1 [[../api-design/signup|signup]] — `UserRepository` (MyBatis)

```java
// infrastructure/persistence/mybatis/user/UserMapper.java
@Mapper
public interface UserMapper {
    Optional<UserRow> findById(@Param("id") String id);
    Optional<UserRow> findByEmailIgnoreCase(@Param("email") String email);
    boolean existsByEmailIgnoreCase(@Param("email") String email);
    int insert(UserRow row);
    int update(UserRow row);
}

// MyBatis Adapter
@Repository
@RequiredArgsConstructor
public class MyBatisUserRepository implements UserRepository {
    private final UserMapper mapper;
    private final Clock clock;

    @Override public boolean existsByEmail(Email email) {
        return mapper.existsByEmailIgnoreCase(email.value());
    }
    @Override public Optional<User> findByEmail(Email email) {
        return mapper.findByEmailIgnoreCase(email.value()).map(this::toDomain);
    }
    @Override public User save(User user) {
        var row = toRow(user, Instant.now(clock));
        var existing = mapper.findById(user.id().value());
        try {
            if (existing.isPresent()) mapper.update(row);
            else mapper.insert(row);
        } catch (DuplicateKeyException e) {
            // PG/MySQL UNIQUE violation
            throw new EmailAlreadyExistsException(user.email());
        }
        return user;
    }
    private User toDomain(UserRow r) {
        return User.reconstitute(
            new UserId(r.id()), new Email(r.email()), new PasswordHash(r.passwordHash()),
            r.name(), UserStatus.valueOf(r.status()), r.createdAt());
    }
    private UserRow toRow(User u, Instant now) { /* ... */ throw new UnsupportedOperationException(); }
}
```

```xml
<!-- mapper/user/UserMapper.xml -->
<mapper namespace="com.example.shop.infrastructure.persistence.mybatis.user.UserMapper">
    <select id="findByEmailIgnoreCase" resultType="UserRow">
        SELECT id, email, password_hash AS passwordHash, name, status, created_at AS createdAt, updated_at AS updatedAt
        FROM users WHERE lower(email) = lower(#{email})
    </select>
    <select id="existsByEmailIgnoreCase" resultType="boolean">
        SELECT EXISTS(SELECT 1 FROM users WHERE lower(email) = lower(#{email}))
    </select>
    <insert id="insert">
        INSERT INTO users (id, email, password_hash, name, status, created_at, updated_at)
        VALUES (#{id}, #{email}, #{passwordHash}, #{name}, #{status}, #{createdAt}, #{updatedAt})
    </insert>
</mapper>
```

MyBatis 포인트:
- JPA dirty checking 없음 → **명시적 `update()`** 호출 필수
- `DuplicateKeyException` (Spring) 으로 DB unique violation 변환
- `UserRow` 는 mapper 전용 DTO — 도메인 (`User`) 이 알지 못함

### 10.5.2 [[../api-design/login-jwt|login-jwt]] — `RefreshTokenRepository` (MyBatis)

```java
@Mapper
public interface RefreshTokenMapper {
    Optional<RefreshTokenRow> findByTokenHash(@Param("hash") String hash);
    Optional<RefreshTokenRow> findById(@Param("id") String id);
    int insert(RefreshTokenRow row);
    int updateStatus(@Param("id") String id, @Param("status") String status,
                     @Param("rotatedToId") String rotatedToId);
    int revokeAllForUser(@Param("userId") String userId);
}
```

```xml
<update id="revokeAllForUser">
    UPDATE refresh_tokens
    SET status = 'REVOKED'
    WHERE user_id = #{userId} AND status = 'ACTIVE'
</update>
```

MyBatis 포인트:
- `revokeAllForUser` 는 명시적 단일 UPDATE — JPA `@Modifying` 보다 가독성 높음

### 10.5.3 [[../api-design/product-crud|product-crud]] — `ProductRepository` + 1:N

가장 어려운 케이스 — Product (Aggregate) 가 options / images / tags 자식 컬렉션을 가짐. JPA cascade 자동 X. **명시적 fetch + 명시적 cascade 저장**.

```java
@Mapper
public interface ProductMapper {
    Optional<ProductRow> findById(@Param("id") String id);
    int insertProduct(ProductRow row);
    int updateProduct(ProductRow row);

    List<ProductOptionRow> findOptionsByProductId(@Param("productId") String productId);
    int insertOption(ProductOptionRow row);
    int deleteOptionsByProductId(@Param("productId") String productId);

    List<ProductImageRow> findImagesByProductId(@Param("productId") String productId);
    int insertImage(ProductImageRow row);
    int deleteImagesByProductId(@Param("productId") String productId);
}

// Adapter — Aggregate 저장의 책임을 통째로 짊어짐
@Repository
public class MyBatisProductRepository implements ProductRepository {
    private final ProductMapper mapper;

    @Override public Optional<Product> findById(ProductId id) {
        var row = mapper.findById(id.value()).orElse(null);
        if (row == null) return Optional.empty();
        var options = mapper.findOptionsByProductId(id.value());
        var images = mapper.findImagesByProductId(id.value());
        return Optional.of(toDomain(row, options, images));
    }

    @Override
    @Transactional
    public Product save(Product product) {
        var existing = mapper.findById(product.id().value()).isPresent();
        if (existing) mapper.updateProduct(toRow(product));
        else          mapper.insertProduct(toRow(product));

        // 자식 cascade — 단순 구현: 전체 교체. 큰 컬렉션은 diff 권장.
        mapper.deleteOptionsByProductId(product.id().value());
        mapper.deleteImagesByProductId(product.id().value());
        product.options().forEach(o -> mapper.insertOption(toOptionRow(o, product.id())));
        product.images().forEach(i -> mapper.insertImage(toImageRow(i, product.id())));

        return product;
    }
}
```

MyBatis 포인트:
- **JPA cascade 없음** — adapter 가 자식 INSERT/DELETE 명시적 처리
- 단순 패턴 = "전체 교체" (작은 컬렉션 OK). 큰 컬렉션은 **diff 후 변경된 것만**
- 트랜잭션 한 묶음 필수 (`@Transactional`)
- 1:N read 는 **별도 select** (cartesian 회피) — 또는 한 쿼리 + `<collection>` ([[mybatis#5.2 1:N]])

### 10.5.4 [[../api-design/product-search|product-search]] — MyBatis 의 진가

MyBatis 가 가장 빛나는 영역. 동적 SQL + cursor + 다중 필터.

```xml
<select id="search" resultMap="ProductSummaryMap">
    SELECT
      p.id, p.name, p.base_price_krw AS basePriceKrw, p.main_image_url AS mainImageUrl,
      p.created_at AS createdAt, b.id AS brandId, b.name AS brandName,
      CASE WHEN COALESCE(SUM(po.stock), 0) = 0 THEN true ELSE false END AS soldOut
    FROM products p
      JOIN brands b ON p.brand_id = b.id
      LEFT JOIN product_options po ON po.product_id = p.id
    <where>
        AND p.status = 'ACTIVE'
        <if test="q != null and q != ''">
            AND lower(p.name) LIKE concat('%', lower(#{q}), '%')
        </if>
        <if test="brand != null and brand.size() > 0">
            AND p.brand_id IN
            <foreach collection="brand" item="b" open="(" separator="," close=")">#{b}</foreach>
        </if>
        <if test="priceMin != null">  AND p.base_price_krw &gt;= #{priceMin} </if>
        <if test="priceMax != null">  AND p.base_price_krw &lt;= #{priceMax} </if>
        <if test="cursor != null">
            AND (p.created_at, p.id) &lt; (#{cursor.lastCreatedAt}, #{cursor.lastId})
        </if>
    </where>
    GROUP BY p.id, b.id, b.name
    ORDER BY p.created_at DESC, p.id DESC
    LIMIT #{limit}
</select>
```

→ JPA Specification + N+1 회피 + 커서를 합친 분량을 **하나의 XML** 로. **공존 모드에서 검색 / 보고서는 MyBatis** 가 표준 이유.

### 10.5.5 [[../api-design/cart|cart]] — `CartRepository` (MyBatis)

JPA 와 동일 패턴. `member` 는 RDB (MyBatis 가능), `guest` 는 Redis (변동 없음).

### 10.5.6 [[../api-design/order-stock|order-stock]] — 비관 락 SQL 직접

```xml
<select id="findOptionForUpdate" resultType="ProductOptionRow">
    SELECT id, name, value, extra_price_krw AS extraPriceKrw, stock
    FROM product_options
    WHERE id = #{id}
    FOR UPDATE
</select>
```

→ JPA `@Lock(PESSIMISTIC_WRITE)` 와 같음. SQL 가독성 ↑.

```xml
<update id="decreaseStock">
    UPDATE product_options
    SET stock = stock - #{qty}
    WHERE id = #{id} AND stock &gt;= #{qty}
</update>
```

→ **rowCount = 0 면 재고 부족** — application 에서 검사. SELECT 없이 한 쿼리로 검증 + 차감 가능 (낙관 락 식).

### 10.5.7 [[../api-design/payment-pg|payment-pg]] — webhook 멱등 + audit

```xml
<insert id="insertWebhook">
    INSERT INTO payment_webhooks (id, provider, event_id, payment_key, event_type, payload, signature, received_at)
    VALUES (#{id}, #{provider}, #{eventId}, #{paymentKey}, #{eventType}, #{payload}::jsonb, #{signature}, #{receivedAt})
    ON CONFLICT (provider, event_id) DO NOTHING
</insert>
```

→ **`ON CONFLICT DO NOTHING`** 으로 멱등을 SQL 한 줄에. JPA 보다 명확.

### 10.5.8 공통 — 모든 MyBatis Adapter 의 5가지 규칙

1. **Row DTO 는 mapper 전용** — 도메인 (`User`) 이 알지 못함. Adapter 가 변환.
2. **`@Transactional` 안에서만 mapper 호출** — connection pool / SqlSession scope.
3. **`#{}` 만 사용** — `${}` 는 메타 (컬럼명 동적) 에만, 사용자 입력은 절대 X.
4. **자식 cascade 는 명시적** — JPA 처럼 자동 X. delete + insert 또는 diff.
5. **map-underscore-to-camel-case = true** + AS 별칭으로 매핑 안정.

---

## 11. 함정 모음

### 함정 1 — `${}` 사용 (SQL injection)
```xml
WHERE id = ${id}                     <!-- ⚠️ SQL injection. 사용자 입력 X -->
WHERE id = #{id}                     <!-- ✅ PreparedStatement parameter -->
```

`${}` 는 **컬럼명 / 테이블명 동적 지정** 같은 메타에만. 사용자 입력은 **무조건 `#{}`**.

### 함정 2 — `<if>` 안에 SQL injection
```xml
<if test="orderBy != null">
    ORDER BY ${orderBy}              <!-- ⚠️ 사용자가 보낸 orderBy 검증 X -->
</if>
```

화이트리스트 검증 후 enum 변환 / 매핑 필요.

### 함정 3 — `<resultMap>` 의 column / property 오타
런타임 에러. 어디서 잘못 됐는지 디버깅 어려움. **테스트 필수**.

### 함정 4 — `@Param` 누락
```java
@Select("WHERE id = #{id}")
User find(String id);                  // OK — 단일 인자
@Select("WHERE id = #{id} AND status = #{status}")
User find(String id, String status);   // ⚠️ #{0}, #{1} 또는 @Param 필요
```

명시적 `@Param("id")` 항상 권장.

### 함정 5 — `map-underscore-to-camel-case = false`
컬럼 `password_hash` → property `passwordHash` 매핑 안 됨. **기본 true**.

### 함정 6 — 1:N 매핑 시 cartesian 의 row 폭발
JPA 와 같은 함정. **별도 select 또는 조절**.

### 함정 7 — `RETURNING` 이 PG / Oracle 만
```sql
INSERT INTO users (...) VALUES (...) RETURNING id
```
MySQL 은 `useGeneratedKeys = true` + `keyProperty = "id"`.

### 함정 8 — 트랜잭션 없이 SqlSession 직접
SqlSession 이 매번 새로 만들어짐 → connection 매번 새로. **`@Transactional`** 안에서.

### 함정 9 — XML 파일이 classpath 에 없음
`mybatis.mapper-locations: classpath:mapper/**/*.xml` 경로 확인. Gradle / Maven 의 resource directory 설정 누락.

### 함정 10 — `<foreach>` 의 너무 큰 컬렉션
`IN (id1, id2, ..., id10000)` = PG / MySQL plan cache 폭발. **chunk** (1000 단위로 쪼개기) 또는 임시 테이블 / 배열.

### 함정 11 — `select count` 별도 호출
페이지네이션 화면에서 N+1. **한 mapper 메서드로 list + count**.

### 함정 12 — Hot reload 후 mapper 안 바뀜
XML 변경 후 재기동 필요 (Spring Boot devtools 가 잡지 못함).

---

## 12. JPA vs MyBatis — 같은 기능의 코드 길이

같은 회원가입 INSERT:

### JPA

```java
@Service @RequiredArgsConstructor
class JpaSignupService {
    private final UserJpaRepository repo;
    @Transactional
    public void save(UserJpaEntity user) { repo.save(user); }   // dirty checking
}
```

### MyBatis

```java
@Service @RequiredArgsConstructor
class MyBatisSignupService {
    private final UserMapper mapper;
    @Transactional
    public void save(UserRow row) { mapper.insert(row); }
}

// + UserMapper.xml 의 <insert id="insert"> 작성
```

→ MyBatis 는 mapper / XML / DTO 가 보이고 명확. JPA 는 짧지만 마법.

---

## 13. 운영 체크리스트

- [ ] `default-statement-timeout` 명시 (3~5초)
- [ ] `cache-enabled = false` (운영은 외부 캐시 사용)
- [ ] `<foreach>` 의 컬렉션 크기 한계 (chunking)
- [ ] PreparedStatement (`#{}`) 만 사용 — `${}` 는 메타에만
- [ ] `@Mapper` 의 메서드 마다 IT 테스트
- [ ] mapper XML 의 `parameterType` / `resultMap` ID 일관성
- [ ] SQL 로그 (p6spy) 같음
- [ ] `@Transactional` 경계 정확히
- [ ] `mybatis-plus` 또는 `mybatis-mapper3` 같은 자동 매핑 도구 검토 (코드 양 줄이기)

---

## 14. 관련

- [[database|↑ database hub]]
- [[jpa]] — JPA 본편 / 비교
- [[jpa-mybatis-coexist]] — 같이 쓸 때
- [[../pitfalls/pitfalls|↗ pitfalls]] — SQL injection / 트랜잭션 함정
- [[../api-design/signup#6.5]] — Adapter 패턴 (JPA 와 동일 구조)
