---
title: "RBAC + Permissions — Role-based / Hierarchical / @PreAuthorize"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - auth
  - rbac
  - authorization
---

# RBAC + Permissions — Role-based / Hierarchical / @PreAuthorize

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | Role + Permission + hierarchical + ownership |

**[[api-design|↑ api-design hub]]**

> 📐 **ORM**: §6 는 JPA Adapter sketch. 공통: [[../common/response-envelope]] · [[../common/security-config]].

---

## 1. 무엇을 만드는가

권한 모델 — endpoint / 도메인 객체 / 필드 단위 접근 제어.

```
USER       → 자기 데이터만
SELLER     → 자기 상품 / 주문 + USER 권한
ADMIN      → 모든 데이터 read / write (특정 모듈 제외)
MASTER     → 모든 권한 + key rotation 등 critical

권한 매트릭스 예 (이커머스):
                              USER  SELLER  ADMIN  MASTER
GET    /products/:id            ✅     ✅      ✅     ✅
POST   /products                ❌     ✅      ✅     ✅
PATCH  /products/:id (own)      ❌     ✅      ✅     ✅
DELETE /products/:id (other's)  ❌     ❌      ✅     ✅
POST   /admin/key/rotate        ❌     ❌      ❌     ✅
```

### 1.1 RBAC vs ABAC vs ACL — 본 레시피의 선택

| 모델 | 정의 | 적합 |
| --- | --- | --- |
| **RBAC** (Role-Based) | role 기반 — USER / SELLER / ADMIN | 일반 SaaS (본 레시피) |
| **ABAC** (Attribute-Based) | 속성 기반 (시간 / IP / 부서 ...) | 복잡 정책 / 정부·금융 |
| **ACL** (Access Control List) | 객체별 user 직접 매핑 | Google Drive 식 공유 |

→ 본 vault 는 **RBAC + ownership check 결합** (가장 흔한 SaaS 패턴).

---

## 2. 도메인

### 2.1 Role / Permission

```java
// domain/auth/Role.java
public enum Role {
    USER,       // 기본
    SELLER,     // 셀러
    ADMIN,      // 운영
    MASTER;     // 슈퍼

    public String asAuthority() { return "ROLE_" + name(); }
}

// domain/auth/Permission.java — 세분화 권한 (선택)
public enum Permission {
    // 상품
    PRODUCT_READ, PRODUCT_WRITE, PRODUCT_DELETE,
    // 주문
    ORDER_READ_OWN, ORDER_READ_ANY, ORDER_REFUND,
    // 사용자
    USER_READ, USER_WRITE, USER_DELETE,
    // 관리
    KEY_ROTATE, AUDIT_READ;

    public String asAuthority() { return "PERM_" + name(); }
}
```

### 2.2 Role → Permission 매핑

```java
// domain/auth/RolePermissions.java
public final class RolePermissions {
    private RolePermissions() {}

    private static final Map<Role, Set<Permission>> MAP = Map.of(
        Role.USER, Set.of(
            Permission.PRODUCT_READ,
            Permission.ORDER_READ_OWN
        ),
        Role.SELLER, Set.of(
            Permission.PRODUCT_READ, Permission.PRODUCT_WRITE,
            Permission.ORDER_READ_OWN
        ),
        Role.ADMIN, Set.of(
            Permission.PRODUCT_READ, Permission.PRODUCT_WRITE, Permission.PRODUCT_DELETE,
            Permission.ORDER_READ_ANY, Permission.ORDER_REFUND,
            Permission.USER_READ, Permission.USER_WRITE,
            Permission.AUDIT_READ
        ),
        Role.MASTER, Set.of(Permission.values())
    );

    public static Set<Permission> of(Role role) { return MAP.getOrDefault(role, Set.of()); }
    public static Set<String> authorities(Role role) {
        var s = new HashSet<String>();
        s.add(role.asAuthority());
        of(role).forEach(p -> s.add(p.asAuthority()));
        return s;
    }
}
```

→ JWT 발급 시 `authorities` = role + permission 들을 모두 포함. Spring Security 가 `ROLE_*`, `PERM_*` 둘 다 처리.

### 2.3 hierarchy (옵션)

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("""
        ROLE_MASTER > ROLE_ADMIN
        ROLE_ADMIN > ROLE_SELLER
        ROLE_SELLER > ROLE_USER
    """);
}

@Bean
public DefaultMethodSecurityExpressionHandler methodSecurityExpressionHandler(RoleHierarchy hierarchy) {
    var handler = new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(hierarchy);
    return handler;
}
```

→ `@PreAuthorize("hasRole('SELLER')")` 가 ADMIN / MASTER 도 통과.

---

## 3. JWT 발급 시 authorities 포함

```java
// JwtTokenProvider 의 generateAccessToken 보강
public String generateAccessToken(String email, Long userId, Role role, String sessionId) {
    var authorities = RolePermissions.authorities(role);     // role + permission String set
    return Jwts.builder()
        .issuer(issuer)
        .subject(email)
        .id(UUID.randomUUID().toString())
        .claim(CLAIM_TYPE, "ACCESS")
        .claim(CLAIM_ID, userId)
        .claim(CLAIM_ROLE, role.asAuthority())
        .claim("authorities", authorities)                    // ⭐ 추가
        .claim(CLAIM_SID, sessionId)
        .issuedAt(Date.from(Instant.now()))
        .expiration(Date.from(Instant.now().plus(accessTtl)))
        .signWith(key, Jwts.SIG.HS256)
        .compact();
}
```

`JwtAuthenticationFilter` 에서:
```java
@SuppressWarnings("unchecked")
var rawAuthorities = (List<String>) claims.get("authorities", List.class);
var grantedAuthorities = rawAuthorities.stream()
    .map(SimpleGrantedAuthority::new)
    .toList();
```

> **함정**: authorities 가 너무 많으면 JWT 가 커짐 (cookie 4KB limit 위험). 50개 이상이면 **server lookup 으로 전환**.

---

## 4. Controller / Service 적용

### 4.1 Endpoint 단 — `@PreAuthorize`

```java
@RestController
@RequestMapping("/api/v1/seller/products")
public class SellerProductController {

    @PreAuthorize("hasRole('SELLER')")             // role 기반
    @PostMapping
    public ResponseEntity<?> create(@Valid @RequestBody CreateProductRequest req, Authentication auth) {
        ...
    }

    @PreAuthorize("hasAuthority('PERM_PRODUCT_DELETE')")    // permission 기반 (더 세밀)
    @DeleteMapping("/{id}")
    public ResponseEntity<?> delete(@PathVariable String id) {
        ...
    }

    @PreAuthorize("hasRole('SELLER') and @productSecurity.canEdit(#id, principal)")  // SpEL + custom
    @PatchMapping("/{id}")
    public ResponseEntity<?> update(@PathVariable String id,
                                     @Valid @RequestBody UpdateProductRequest req,
                                     Authentication auth) {
        ...
    }
}
```

### 4.2 ownership — custom SpEL bean

```java
// presentation/security/ProductSecurity.java
@Component("productSecurity")
@RequiredArgsConstructor
public class ProductSecurity {

    private final ProductRepository products;

    public boolean canEdit(String productId, Object principal) {
        if (!(principal instanceof User user)) return false;
        if (user.role() == Role.ADMIN || user.role() == Role.MASTER) return true;
        return products.findById(new ProductId(productId))
            .map(p -> p.isOwnedBy(user.id()))
            .orElse(false);
    }

    public boolean canDelete(String productId, Object principal) {
        // 셀러는 자기 상품 soft delete 만, ADMIN+ 는 hard delete 가능
        return canEdit(productId, principal);
    }
}
```

→ `@PreAuthorize` 의 SpEL 에서 `@productSecurity.canEdit(#id, principal)` 호출 가능.

### 4.3 Service 단 — 도메인 검증 (이중)

```java
@Service
public class UpdateProductUseCase {
    @Transactional
    public void handle(ProductId id, UserId currentUser, Role currentRole, ...) {
        var product = products.findById(id).orElseThrow(...);

        if (!product.isOwnedBy(currentUser) && currentRole != Role.ADMIN && currentRole != Role.MASTER) {
            throw new BusinessException(ResponseCode.FORBIDDEN, "본인의 상품만 수정 가능합니다.");
        }
        // ...
    }
}
```

> **규칙**: Controller `@PreAuthorize` + Service 도메인 체크 **이중**. Controller 만 의존하면 Service 직접 호출 (배치 등) 시 우회 가능.

---

## 5. Method security 활성

```java
// SecurityConfig 의 @EnableMethodSecurity 가 prePostEnabled = true
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig { ... }
```

여기에 추가:

```java
@Bean
public MethodSecurityExpressionHandler methodSecurityExpressionHandler(RoleHierarchy roleHierarchy) {
    var handler = new DefaultMethodSecurityExpressionHandler();
    handler.setRoleHierarchy(roleHierarchy);
    return handler;
}
```

---

## 6. Permission 동적 변경 — Role 만 vs 동적 Permission

**단순 (Role 만)**: role 변경 시 DB 만 update → 다음 로그인 / 토큰 refresh 후 반영. 본 레시피 권장.

**동적 (Permission 직접 부여)**: `user_permissions` 별도 테이블. 매 요청마다 DB lookup or 캐시.

```sql
-- 옵션: 사용자별 추가 permission
CREATE TABLE user_permissions (
    user_id      CHAR(26) NOT NULL REFERENCES users(id),
    permission   VARCHAR(50) NOT NULL,
    granted_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    granted_by   CHAR(26) NOT NULL REFERENCES users(id),
    PRIMARY KEY (user_id, permission)
);
```

→ Authorities 계산 시 `RolePermissions.authorities(role) ∪ user_permissions` 합집합.

---

## 7. 운영 / 감사

### 7.1 권한 변경 감사 로그

```sql
CREATE TABLE role_change_history (
    id           CHAR(26) PRIMARY KEY,
    user_id      CHAR(26) NOT NULL,
    old_role     VARCHAR(20),
    new_role     VARCHAR(20) NOT NULL,
    changed_by   CHAR(26) NOT NULL,
    reason       TEXT,
    changed_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

→ ADMIN 으로 승격된 사용자는 모두 추적. 사고 시 root cause.

### 7.2 SecurityConfig 의 endpoint 매트릭스 검토

PR 마다 SecurityConfig 의 `requestMatchers(...).hasRole(...)` 행 확인. 새 endpoint 추가 시 정책 누락 방지.

---

## 8. 함정 모음

### 함정 1 — `@PreAuthorize("hasRole('USER')")` 만 신뢰
Service 직접 호출 / 다른 모듈에서 우회 가능. **도메인 객체 단 ownership 체크 이중**.

### 함정 2 — `hasRole('ROLE_USER')` vs `hasRole('USER')`
Spring 은 자동으로 `ROLE_` prefix 붙임. `hasRole('USER')` 만 사용 — `ROLE_USER` 중복.

### 함정 3 — JWT authorities 너무 큼
permission 50개 = JWT 4KB+ . 쿠키 limit. **role 만 토큰, permission 은 server lookup** 또는 cached.

### 함정 4 — role 변경이 토큰에 즉시 반영 X
사용자가 SELLER → ADMIN 승격 후에도 옛 token 은 옛 권한. **session invalidation + refresh** 강제.

### 함정 5 — ownership 체크 누락
`@PreAuthorize("hasRole('SELLER')")` 만 — 다른 셀러 데이터 변경 가능. **`@productSecurity.canEdit(#id, principal)`** 추가.

### 함정 6 — role hierarchy 미사용
ADMIN 이 USER endpoint 호출 시 401. `RoleHierarchy` Bean 으로 자동 상속.

### 함정 7 — 권한 정보 lookup 매번 DB
요청마다 user_permissions 조회 = 부하. **JWT claim + 짧은 TTL 캐시**.

### 함정 8 — MASTER 가 모든 endpoint 통과
critical operations (key rotation 등) 에 추가 인증 (2FA, IP 화이트리스트) 권장.

### 함정 9 — `@Secured("ROLE_ADMIN")` vs `@PreAuthorize`
`@Secured` 는 단순. `@PreAuthorize` 는 SpEL — 본 vault 는 `@PreAuthorize` 권장.

### 함정 10 — 권한 없을 때 403 vs 404
권한 없는 데이터 존재 여부 노출 (404 = 없음, 403 = 권한 없음) — enumeration. **민감 도메인은 404 통일**.

---

## 9. 운영 체크리스트

- [ ] `@EnableMethodSecurity(prePostEnabled = true)`
- [ ] Role 변경 시 audit log
- [ ] critical role (ADMIN, MASTER) 부여 시 2FA / 추가 검증
- [ ] JWT 의 authorities 크기 모니터 (4KB 미만)
- [ ] PR review 시 SecurityConfig `requestMatchers` 변경 확인
- [ ] 도메인 ownership 체크 — Service / Repository 단도 이중 검증
- [ ] role hierarchy bean 등록 시 `MethodSecurityExpressionHandler` 도

---

## 10. 관련

- [[login-jwt]] — JWT 의 authorities claim
- [[product-crud]] — seller ownership 적용 예
- [[../common/security-config]] — `@EnableMethodSecurity`
- [[two-factor-auth]] — ADMIN 권한 + 2FA
- [[api-design|↑ api-design hub]]
