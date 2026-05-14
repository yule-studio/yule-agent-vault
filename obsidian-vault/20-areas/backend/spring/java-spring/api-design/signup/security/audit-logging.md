---
title: "Audit Logging — 가입 / 로그인 / 비번 변경"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T05:40:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - security
  - audit
  - logging
---

# Audit Logging — 가입 / 로그인 / 비번 변경

**[[security|↑ security hub]]**

> 누가 언제 어디서 무엇을 — 보안 사고 / 분쟁 / GDPR 의 필수.
> 부실하면 **사고 후 영향 분석 불가, 법적 입증 X, 내부자 도용 추적 X**.

---

## 1. 본 vault 결정

- 별도 `auth_audit_log` 테이블 + structured logging (CloudWatch / Sentry).
- 모든 보안 이벤트 (signup / login / failure / password change / 2FA / admin 작업) 기록.
- 보유 기간 — 1년 (한국 정보통신망법) + S3 archive (5년).

---

## 2. 무엇을 기록 (Event 종류)

| Event | 트리거 | 보유 기간 |
| --- | --- | --- |
| SIGNUP | 회원가입 | 1년 |
| SIGNUP_FAILED | 검증 실패 (이메일 중복 등) | 30일 |
| LOGIN_SUCCESS | 로그인 성공 | 1년 |
| LOGIN_FAILED | 로그인 실패 | 1년 (brute force 추적) |
| LOGOUT | 명시적 logout | 30일 |
| PASSWORD_CHANGED | 비번 변경 | 5년 |
| PASSWORD_RESET_REQUESTED | reset 메일 발송 | 30일 |
| PASSWORD_RESET_CONFIRMED | reset 완료 | 5년 |
| EMAIL_VERIFICATION | 인증 완료 | 1년 |
| 2FA_ENABLED / DISABLED | 2FA 설정 변경 | 5년 |
| ROLE_CHANGED | admin 권한 부여 / 강등 | 5년 |
| TOKEN_REVOKED | refresh revoke (자체 / 강제) | 1년 |
| SUSPICIOUS_ACTIVITY | reuse detection / 의심 IP | 5년 |
| GDPR_DATA_EXPORT | 사용자 데이터 export | 5년 |
| GDPR_DELETION_REQUESTED | 탈퇴 요청 | 5년 |

---

## 3. 무엇을 기록 (필드)

```sql
CREATE TABLE auth_audit_log (
    id            CHAR(26) PRIMARY KEY,
    event_type    VARCHAR(50) NOT NULL,          -- LOGIN_SUCCESS / ...
    user_id       CHAR(26),                       -- nullable (anonymous event)
    target_user_id CHAR(26),                       -- admin action 의 대상
    actor_role    VARCHAR(20),                    -- USER / ADMIN / SYSTEM
    ip_address    VARCHAR(45),
    user_agent    VARCHAR(500),
    request_id    VARCHAR(50),                    -- trace_id
    success       BOOLEAN NOT NULL,
    failure_reason VARCHAR(200),
    metadata      JSONB,                          -- event 별 추가 정보
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),

    INDEX ix_audit_user_event (user_id, event_type, created_at DESC),
    INDEX ix_audit_event_time (event_type, created_at DESC)
);
```

### 3.1 각 필드의 "왜"

**event_type**
- 분류 / 검색 / 메트릭 기준.
- enum 외 값 통과 차단 (CHECK 옵션).

**user_id (nullable)**
- LOGIN_FAILED 시 user 식별 안 됐을 수 있음 (잘못된 email).
- SIGNUP_FAILED 도 마찬가지.

**target_user_id**
- admin 이 user 정지 시 → actor = admin, target = user.
- 동일 user 의 self-action 은 target_user_id NULL.

**actor_role**
- USER (본인) / ADMIN (관리자) / SYSTEM (자동 cleanup / scheduled job).
- 권한 audit.

**ip_address / user_agent**
- 의심 활동 분석.
- 분쟁 시 "어디서 했나" 입증.

**request_id (trace_id)**
- 분산 추적 — application log 와 매핑.
- 같은 요청의 여러 event 연결.

**failure_reason**
- LOGIN_FAILED 시 "PASSWORD_MISMATCH" / "USER_LOCKED" / "USER_NOT_FOUND".
- brute force 패턴 분석.

**metadata JSONB**
- event 별 추가 정보 (구조 다양 — JSONB).
- 예: PASSWORD_CHANGED 의 `{ method: "self" / "admin-reset" }`.

---

## 4. 구현

### 4.1 Service

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class AuthAuditLogger {

    private final AuthAuditLogRepository repo;
    private final HttpServletRequest request;        // ip / user-agent 추출

    @Transactional(propagation = Propagation.REQUIRES_NEW)   // 비즈니스 영향 X
    public void log(AuthAuditEvent event) {
        var entity = new AuthAuditLogEntity(
            UlidCreator.getMonotonicUlid().toString(),
            event.type().name(),
            event.userId() != null ? event.userId().value() : null,
            event.targetUserId() != null ? event.targetUserId().value() : null,
            event.actorRole(),
            extractIp(request),
            request.getHeader("User-Agent"),
            MDC.get("traceId"),
            event.success(),
            event.failureReason(),
            event.metadata(),
            Instant.now()
        );
        repo.save(entity);

        // structured logging — CloudWatch / Sentry
        log.info("audit_event",
            kv("event", event.type()),
            kv("user", event.userId()),
            kv("success", event.success()),
            kv("ip", extractIp(request))
        );
    }
}
```

**왜 `REQUIRES_NEW`**
- audit log 의 트랜잭션이 비즈니스 트랜잭션과 분리.
- 비즈니스 rollback 시 audit 은 보존.
- 비즈니스 commit 후 audit 실패해도 비즈니스 영향 X.

### 4.2 Application 단

```java
@PostMapping("/auth/login")
public LoginResponse login(@Valid @RequestBody LoginRequest req) {
    try {
        var result = authService.login(req.email(), req.password());
        auditLogger.log(AuthAuditEvent.loginSuccess(result.userId()));
        return result;
    } catch (InvalidCredentialsException e) {
        auditLogger.log(AuthAuditEvent.loginFailed(req.email(), "PASSWORD_MISMATCH"));
        throw e;
    } catch (UserNotFoundException e) {
        auditLogger.log(AuthAuditEvent.loginFailed(req.email(), "USER_NOT_FOUND"));
        throw e;
    }
}
```

---

## 5. structured logging (CloudWatch / Sentry)

### 5.1 왜 둘 다 (DB + structured log)

- DB = SQL query / 분쟁 / audit (relational).
- structured log = real-time alarm + 분석 (Athena / Kibana).

### 5.2 형식

```json
{
  "@timestamp": "2026-05-15T03:00:00.000Z",
  "level": "INFO",
  "event": "audit_event",
  "type": "LOGIN_FAILED",
  "user_id": null,
  "email_masked": "a***e@example.com",
  "ip": "1.2.3.4",
  "user_agent": "Mozilla/5.0 ...",
  "request_id": "01HQ...",
  "success": false,
  "reason": "PASSWORD_MISMATCH"
}
```

### 5.3 Logback config

```xml
<appender name="STRUCTURED" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <customFields>{"app":"yule-studio","env":"prod"}</customFields>
    </encoder>
</appender>
```

---

## 6. 알람 / 모니터링

| 메트릭 | 알람 |
| --- | --- |
| LOGIN_FAILED rate | baseline 의 5배 ↑ = brute force 의심 |
| LOGIN_FAILED 같은 IP | 분당 100+ = brute force |
| SUSPICIOUS_ACTIVITY count | 1시간 5+ = 도난 의심 |
| ADMIN_ACTION count | 1시간 baseline ↑ = 내부자 도용 |
| GDPR_DELETION rate | 평소의 10배 = 사고 / 광고 backlash |

---

## 7. 보유 기간 / Cleanup

```sql
-- 30일 지난 SIGNUP_FAILED / LOGOUT 삭제
DELETE FROM auth_audit_log
WHERE event_type IN ('SIGNUP_FAILED', 'LOGOUT', 'PASSWORD_RESET_REQUESTED')
  AND created_at < now() - INTERVAL '30 days';

-- 1년 지난 일반 event 도 삭제 (또는 S3 archive)
DELETE FROM auth_audit_log
WHERE event_type IN ('SIGNUP', 'LOGIN_SUCCESS', 'LOGIN_FAILED', 'EMAIL_VERIFICATION', 'TOKEN_REVOKED')
  AND created_at < now() - INTERVAL '1 year';

-- 5년 지난 critical event 도 삭제 (또는 S3 archive)
DELETE FROM auth_audit_log
WHERE event_type IN ('PASSWORD_CHANGED', '2FA_ENABLED', 'ROLE_CHANGED')
  AND created_at < now() - INTERVAL '5 years';
```

### 7.1 왜 기간별 분리

- 모든 event 5년 보존 = DB 폭증.
- critical (분쟁 위험 큰) 만 5년.
- 단순 (signup failed) 30일 충분.

### 7.2 S3 archive

```
[Daily batch]
   SELECT * FROM auth_audit_log WHERE created_at < now() - INTERVAL '1 year'
   ↓
   parquet / gzip 파일로 S3 export
   ↓
   DB 삭제
```

**왜**
- S3 = 저렴 ($0.023 / GB / month vs RDS 비싸).
- Athena 로 SQL 분석 가능.
- 한국 정보통신망법 / GDPR 의 장기 보존 의무.

---

## 8. 함정 모음

### 함정 1 — audit log 안 함
사고 시 영향 분석 X. 분쟁 시 입증 X.
→ 모든 보안 이벤트 기록.

### 함정 2 — log 만 (DB 없이)
log aggregator 손실 / 쿼리 어려움.
→ DB + structured log 둘 다.

### 함정 3 — `failure_reason` 누락
brute force 패턴 분석 X.
→ 명시 (PASSWORD_MISMATCH / USER_LOCKED / USER_NOT_FOUND).

### 함정 4 — audit 가 비즈니스 트랜잭션 안
비즈니스 rollback 시 audit 도 사라짐.
→ REQUIRES_NEW.

### 함정 5 — IP 추출 잘못
모든 사용자가 ELB IP.
→ X-Forwarded-For + forward-headers-strategy.

### 함정 6 — 평문 PII 저장
email / phone 평문 = GDPR 위반.
→ 마스킹.

### 함정 7 — 보유 기간 없음
DB 무한 증가.
→ event 별 기간 + cleanup.

### 함정 8 — 알람 없음
brute force 감지 X.
→ 메트릭 + 알람.

### 함정 9 — admin action 기록 안 함
내부자 도용 추적 X.
→ ADMIN_ACTION 명시.

### 함정 10 — `request_id` (trace_id) 누락
같은 요청의 여러 event 연결 X.
→ MDC + trace_id 전파.

### 함정 11 — metadata 가 nullable 인데 검색 어려움
JSONB 안의 특정 키 검색 풀스캔.
→ JSONB 인덱스 또는 자주 검색하는 필드는 별도 컬럼.

### 함정 12 — SUSPICIOUS_ACTIVITY 정의 없음
"의심 활동" 추적 누락.
→ reuse detection / 의심 IP / 평소와 다른 디바이스 → 명시 event.

---

## 9. GDPR 의 right to be informed

```
사용자가 "내 데이터 보여줘" (Right of access) 요청 시:
→ auth_audit_log 의 자기 user_id row 제공
→ "어떤 event 가 언제 발생했나" 사용자에 보고

사용자가 "삭제해줘" (Right to erasure) 시:
→ user 데이터 삭제 (soft delete + anonymize)
→ auth_audit_log 의 user_id 도 anonymize (NULL or hash)
→ 단 audit 자체는 보존 (법적 의무)
```

**왜 audit 는 보존**
- 법적 의무 (1년+).
- 단 user 식별 정보 제거 = anonymization.

---

## 10. 관련

- [[security|↑ security hub]]
- [[sensitive-data-handling]] — PII 마스킹
- [[attack-defense]] — brute force / suspicious 활동
- [[../database/encryption-at-rest]] — audit log 도 보호
- [[../signup-impl]] · [[../login-impl]] — 통합
- 외부 — GDPR Right of access, 한국 정보통신망법 보유 기간
