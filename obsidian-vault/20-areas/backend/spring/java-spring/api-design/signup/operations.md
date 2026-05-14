---
title: "signup §9 — 운영 체크리스트"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T23:50:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - signup
  - operations
---

# signup §9 — 운영 체크리스트

**[[signup|↑ signup hub]]**  ·  ← [[testing]]  ·  → [[implementation-order]]

---

## 1. 배포 전 — 환경변수 / 설정값

| 변수 | 의미 | 필수 |
| --- | --- | --- |
| `DB_USER`, `DB_PASSWORD` | PostgreSQL 인증 | ✅ |
| `DB_HOST`, `DB_PORT`, `DB_NAME` | 연결 정보 | ✅ |
| `JWT_SECRET` | (login 의존) | login-jwt 에서 |
| `CORS_ORIGINS` | 허용 origin (csv) | ✅ |
| `APP_PROFILE` | `local / dev / staging / prod` | ✅ |
| `SECRET_ENCRYPTION_KEY` | 컬럼 암호화 (옵션) | △ |
| `HIBP_API_BASE` | 유출 패스워드 체크 (옵션) | △ |
| `OUTBOX_WORKER_ENABLED` | 워커 활성 | ✅ |
| `EMAIL_FROM` | 메일 발신자 | ✅ |

**Vault 주입**:
```yaml
spring:
  config:
    import: "vault://"        # Spring Cloud Vault 또는 AWS Secrets Manager
```

---

## 2. 로그

### 2.1 로그 레벨 (운영)

```yaml
logging:
  level:
    root: INFO
    com.example.shop: INFO
    org.springframework.security: INFO
    org.hibernate.SQL: WARN                # 운영 DEBUG X
    org.hibernate.orm.jdbc.bind: WARN
    org.springframework.transaction: INFO
```

### 2.2 마스킹 / 노출 금지

| 항목 | 처리 |
| --- | --- |
| password 평문 | `SignupRequest.toString()` 마스킹 (`***`) |
| password hash | log 출력 X (SQL log 도 WARN 이상) |
| email | 일부 마스킹 권장 (`a***e@x.com`) — DEBUG 외 |
| 응답 body 통째 로깅 X | ApiLogFilter 가 status / latency 만 |

### 2.3 logback / structured

```xml
<!-- logback-spring.xml — JSON 형식 (ELK / Datadog) -->
<appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <includeMdcKeyName>userId</includeMdcKeyName>
        <includeMdcKeyName>requestId</includeMdcKeyName>
    </encoder>
</appender>
```

**MDC** 로 `requestId`, `userId` 자동 부여 — `ApiLogFilter`.

---

## 3. 메트릭 (Prometheus / Micrometer)

| 메트릭 | 타입 | 알람 임계 |
| --- | --- | --- |
| `signup.attempts.total{result="success"}` | Counter | — |
| `signup.attempts.total{result="duplicate"}` | Counter | 갑작스러운 spike = brute force |
| `signup.attempts.total{result="weak_password"}` | Counter | — |
| `signup.attempts.total{result="error"}` | Counter | 5% 이상 |
| `signup.latency` | Histogram | p95 > 500ms |
| `password.hash.duration` | Histogram | argon2 비용 모니터 |
| `email_outbox.pending.gauge` | Gauge | > 1000 = 워커 이상 |
| `db.connection.pool.active` | Gauge | > 80% pool size |

```java
@Service
@RequiredArgsConstructor
public class SignupUseCase {

    private final MeterRegistry meter;

    @Transactional
    public User handle(SignupCommand cmd) {
        var sample = Timer.start(meter);
        try {
            var user = doHandle(cmd);
            meter.counter("signup.attempts.total", "result", "success").increment();
            return user;
        } catch (EmailAlreadyExistsException e) {
            meter.counter("signup.attempts.total", "result", "duplicate").increment();
            throw e;
        } catch (BusinessException e) {
            meter.counter("signup.attempts.total", "result", e.getResponseCode().code()).increment();
            throw e;
        } catch (Exception e) {
            meter.counter("signup.attempts.total", "result", "error").increment();
            throw e;
        } finally {
            sample.stop(meter.timer("signup.latency"));
        }
    }
}
```

---

## 4. 알림

| 시그널 | 채널 | 우선순위 |
| --- | --- | --- |
| 5xx 비율 > 5% | PagerDuty (즉시) | P1 |
| signup 실패율 > 30% | Slack #alerts | P2 |
| 이메일 outbox pending > 1000 | Slack (워커 점검) | P2 |
| password.hash.duration p95 > 1s | Slack | P3 |
| DB connection pool 고갈 | PagerDuty | P1 |
| 같은 IP 5분 50회 signup | Slack #security | P2 (봇 의심) |

---

## 5. 롤백

### 5.1 코드 롤백 (앱 버전)

- Blue-green / Canary 배포 — 신버전 5% 에 release → 메트릭 정상 시 100%
- 5xx 비율 / latency 비정상 = 자동 롤백 (Argo Rollouts / Flagger)

### 5.2 DB 스키마 롤백

| 변경 종류 | 롤백 |
| --- | --- |
| 새 컬럼 추가 (nullable) | drop column (data loss 가능) |
| 새 컬럼 추가 (not null + default) | drop column |
| 컬럼 이름 변경 | 새 이름 + 옛 컬럼 dual write (점진적) |
| 컬럼 type 변경 | 새 컬럼 + migration + 옛 컬럼 drop |
| 컬럼 삭제 | 위 reverse |

→ **Flyway 의 V 마이그레이션은 forward only**. 롤백은 새 V 추가 (`V99__rollback_xyz.sql`).

### 5.3 코드 vs 스키마 호환

- 코드 v1 (옛 schema) 가 schema v2 (새 컬럼) 호환되어야 함
- v1 → v2 배포 순서: schema → code (1 단계씩, 옛 코드와 새 schema 가 호환)

---

## 6. 장애 포인트

| 영역 | 장애 가능성 |
| --- | --- |
| DB connection pool 고갈 | 동시 가입 폭주 + connection leak |
| argon2 CPU 폭증 | DDoS 식 가입 시도 → 모든 가입 느려짐 |
| 이메일 outbox 워커 다운 | 인증 메일 미발송 → 사용자 항의 |
| HIBP API 외부 호출 | 외부 다운 시 가입 막힘 (timeout / fail-open) |
| Flyway 마이그레이션 실패 | 앱 시작 X — 알람 |
| password_hash 컬럼 길이 부족 | argon2 PHC 가 더 길어지면 truncate → 로그인 실패 |

---

## 7. 운영 직전 체크리스트

- [ ] DB UNIQUE 인덱스 `ux_users_email` 존재
- [ ] `password_hash VARCHAR(255)` 충분한지 (argon2 PHC 가 ~120 자 → OK)
- [ ] `application.yml` 의 `spring.jpa.open-in-view: false`
- [ ] `JwtAuthenticationEntryPoint` 가 SC_UNAUTHORIZED (401) 응답 (원본 버그 주의 — [[../../common/security-config#4.1]])
- [ ] HTTPS / HSTS / TLS 1.2+ 강제
- [ ] CORS allowedOrigins 명시
- [ ] WAF / CAPTCHA (대량 가입 봇 대비)
- [ ] Rate limit (Bucket4j) 적용 — IP 별 1시간 3회
- [ ] 이메일 outbox 워커 라이브니스 (kubernetes liveness probe)
- [ ] Prometheus / Grafana dashboard 의 signup 메트릭
- [ ] 알람 PagerDuty / Slack 라우팅
- [ ] 침해 사고 대응 절차 (모든 사용자 패스워드 강제 재설정 가능 — DB UPDATE + 알림)
- [ ] 개인정보 백업 / 암호화 (RDS encryption at rest)
- [ ] 감사 로그 (`role_change_history` 등 — RBAC 에서)

---

## 8. 배포 후 검증 (smoke test)

```bash
# 1. health check
curl https://api.example.com/actuator/health

# 2. 새 가입
curl -X POST https://api.example.com/api/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d '{
    "email": "smoke-$(date +%s)@example.com",
    "password": "Tr0ub4dor!12",
    "name": "Smoke Test",
    "termsAgreed": true
  }'

# 3. Swagger UI 가 차단되었는지 (운영)
curl https://api.example.com/swagger-ui/index.html
# 기대: 404 or 401 (block)

# 4. metric scrape
curl https://api.example.com/actuator/prometheus | grep signup
```

배포 후 첫 30분:
- 5xx 모니터링
- p95 latency 추세
- 이메일 발송 워커 큐 길이

---

## 9. 관련

- [[signup|↑ signup hub]]
- [[testing]] — 이전 (§8)
- [[implementation-order]] — 다음 (§10)
- [[pitfalls]] — 함정 모음 (§11)
- [[../../common/security-config]] — JwtAuthenticationEntryPoint 버그
