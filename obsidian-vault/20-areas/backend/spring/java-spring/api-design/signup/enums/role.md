---
title: "Role — user 의 권한 등급"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T01:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - enum
  - role
  - rbac
---

# Role — user 의 권한 등급

**[[enums|↑ enums hub]]**  ·  관련: [[../../rbac-permissions]]

> RBAC 의 기초. 4개 + 상속 계층. JWT claim 의 `role` 도 이 값.

---

## 1. 값 정의

```java
// src/main/java/com/example/shop/domain/user/Role.java
public enum Role {

    /** 기본 사용자. */
    USER,

    /** 셀러 (상품 등록 / 수정). */
    SELLER,

    /** 운영자 (사용자 / 콘텐츠 관리). */
    ADMIN,

    /** 슈퍼유저 (시스템 설정 / key rotation). */
    MASTER;

    /** Spring Security 표준 — `ROLE_*` prefix */
    public String asAuthority() { return "ROLE_" + name(); }

    public boolean isPrivileged() { return this == ADMIN || this == MASTER; }

    public boolean canImpersonate() { return this == MASTER; }
}
```

---

## 2. 왜 4개

### 2.1 USER

- 모든 가입자의 default
- 자기 데이터만 접근

### 2.2 SELLER

- 일반 user + 상품 등록 / 수정 권한
- 자기 상품 / 주문만 (cross-seller 격리)
- 본 vault: 이커머스 기준. 다른 도메인이면 다른 이름 (`AUTHOR` / `CREATOR` / `INSTRUCTOR` 등)

### 2.3 ADMIN

- 사용자 관리 / 콘텐츠 moderation
- 운영팀이 일상 사용
- 모든 데이터 read / write (특정 critical 제외)

### 2.4 MASTER

- 시스템 설정 (key rotation, secret 변경)
- 단일 또는 소수 인원만
- 평소엔 비활성, 필요 시만 활성 (2FA 필수 + IP 화이트리스트 권장)

---

## 3. 왜 더 많지 / 적지 않은가

### 후보: `MODERATOR` (USER 와 ADMIN 사이)

- 콘텐츠 신고 처리 / 댓글 삭제
- ADMIN 보다 권한 좁음

→ **사업 규모** 에 따라 추가. 일반 SaaS 초기엔 ADMIN 으로 충분.

### 후보: `GUEST` (가입 전)

- anonymous 접근
- enum 보단 **null 또는 Spring Security `anonymousAuthenticationToken`** 으로 표현

→ enum 에 안 넣음.

### 후보: ADMIN 만 (MASTER 없음)

- 모든 critical operation 도 ADMIN
- 사고 위험 (ADMIN 한 명이 시스템 전체 조작 가능)
- **MASTER 분리** = 책임 분산 + 감사

→ MASTER 필요.

---

## 4. 상속 / Hierarchy

```
ROLE_MASTER > ROLE_ADMIN > ROLE_SELLER > ROLE_USER
```

Spring Security `RoleHierarchy`:

```java
@Bean
public RoleHierarchy roleHierarchy() {
    return RoleHierarchyImpl.fromHierarchy("""
        ROLE_MASTER > ROLE_ADMIN
        ROLE_ADMIN > ROLE_SELLER
        ROLE_SELLER > ROLE_USER
    """);
}
```

→ `@PreAuthorize("hasRole('USER')")` 가 SELLER / ADMIN / MASTER 도 통과.

> **주의**: hierarchy 가 의도와 다른 경우 (예: ADMIN 이 SELLER 의 상품 직접 등록 X) — hierarchy 빼고 명시적 매트릭스. 본 vault 는 hierarchy 사용.

---

## 5. Role 변경 정책

| transition | 누가 | 조건 |
| --- | --- | --- |
| USER → SELLER | 셀러 신청 + 관리자 승인 | seller_application 별도 흐름 |
| SELLER → USER | 셀러 자격 박탈 | 부정 사용 시 ADMIN |
| USER → ADMIN | MASTER 만 | 2FA + audit log 필수 |
| ADMIN → MASTER | 외부 system 또는 MASTER 만 | 보통 1명 |

→ Role 변경 시 **반드시 audit log**:

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

---

## 6. JWT claim 의 Role

```java
public String generateAccessToken(String email, Long userId, Role role, String sessionId) {
    return Jwts.builder()
        ...
        .claim("role", role.asAuthority())          // "ROLE_USER" 형태
        ...
}
```

JWT payload:
```json
{
  "sub": "alice@x.com",
  "id": 123,
  "role": "ROLE_USER",
  "sid": "...",
  "exp": 1715688000
}
```

→ JwtAuthenticationFilter 가 SecurityContext 에 set:

```java
var authorities = List.<GrantedAuthority>of(
    new SimpleGrantedAuthority(role)        // role = "ROLE_USER"
);
```

자세히: [[../../common/security-config#3 JwtAuthenticationFilter]].

---

## 7. Role 변경의 즉시 반영 문제

```
사용자 A 가 ROLE_USER 로 로그인 → access token claim "role"="ROLE_USER"
관리자가 A 를 ROLE_SELLER 로 승격 → DB 갱신
하지만 A 의 access token 은 여전히 ROLE_USER
```

해결:
1. **access 짧게 (15분)** — refresh 시 새 token claim 갱신
2. **session UUID (sid) revoke** — 강제 로그아웃 + 재로그인
3. **JWT 검증 시 DB 조회로 role 재확인** — stateless 의 장점 사라짐 (안 권장)

본 vault: **1번** (대부분의 경우). 즉시 반영 critical 한 경우 **2번**.

---

## 8. 권한 매트릭스 (예시)

| Endpoint | USER | SELLER | ADMIN | MASTER |
| --- | --- | --- | --- | --- |
| `GET /products/:id` | ✅ | ✅ | ✅ | ✅ |
| `POST /seller/products` | ❌ | ✅ | ✅ | ✅ |
| `PATCH /seller/products/:id` (자기 것) | ❌ | ✅ | ✅ | ✅ |
| `PATCH /seller/products/:id` (남 것) | ❌ | ❌ | ✅ | ✅ |
| `POST /orders` | ✅ | ✅ | ✅ | ✅ |
| `POST /admin/users/:id/suspend` | ❌ | ❌ | ✅ | ✅ |
| `POST /admin/key/rotate` | ❌ | ❌ | ❌ | ✅ |

→ [[../../rbac-permissions]] 의 권한 모델 상세.

---

## 9. DB

```sql
ALTER TABLE users ADD COLUMN role VARCHAR(20) NOT NULL DEFAULT 'USER';

-- CHECK constraint
ALTER TABLE users ADD CONSTRAINT chk_user_role
    CHECK (role IN ('USER', 'SELLER', 'ADMIN', 'MASTER'));
```

```java
@Enumerated(EnumType.STRING)
@Column(nullable = false, length = 20)
private Role role;
```

---

## 10. 함정 모음

### 함정 1 — `ROLE_USER` 직접 String 비교
`role.equals("USER")` vs `role.equals("ROLE_USER")` 혼동. **enum 사용** 또는 `.asAuthority()` 일관.

### 함정 2 — `hasRole('ROLE_USER')` (SpEL)
Spring 이 자동으로 `ROLE_` prefix 붙임. `hasRole('USER')` 만 사용. 중복은 `ROLE_ROLE_USER` 만들 수 있음.

### 함정 3 — Role 변경 즉시 반영 가정
access token TTL 까지 옛 권한 살아있음. 짧은 TTL + session 강제 로그아웃 옵션.

### 함정 4 — `@PreAuthorize` 만 의존
Service 직접 호출 (배치 / 다른 서비스) 시 우회. **도메인 / Service 단도 ownership 체크**.

### 함정 5 — MASTER 의 패스워드 / 2FA 부실
시스템 전체 위험. MASTER 는 **하드웨어 키 / 2FA + IP 화이트리스트** 강제.

### 함정 6 — Role hierarchy 의 의도하지 않은 통과
"USER endpoint" 인데 ADMIN 도 OK = 의도. "SELLER 만 endpoint" 인데 ADMIN 통과 = 의도? 정책 명시.

### 함정 7 — Role 변경 audit 없음
사고 시 root cause 추적 불가. **role_change_history 별도 테이블**.

### 함정 8 — DELETED user 의 role
DELETED user 가 JWT 유효한 동안 권한 사용 가능. **JwtAuthenticationFilter 가 status 검사**.

---

## 11. 관련

- [[enums|↑ enums hub]]
- [[../../rbac-permissions]] — Permission enum + 권한 매트릭스
- [[../../common/security-config]] — `@EnableMethodSecurity` + RoleHierarchy
- [[user-status]] — DELETED user 의 권한 무효화
