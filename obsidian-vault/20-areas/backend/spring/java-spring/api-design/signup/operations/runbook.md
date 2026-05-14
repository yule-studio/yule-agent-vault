---
title: "Runbook — 장애 대응 / Smoke Test / 체크리스트"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:35:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - operations
  - runbook
  - incident-response
---

# Runbook — 장애 대응 / Smoke Test / 체크리스트

**[[operations|↑ operations hub]]**

> "알람 받았을 때 무엇을 어떤 순서로 하나" — 명문화된 절차.
> 부실하면 **사고 시 우왕좌왕 → 복구 늦어짐 → 사용자 영향 ↑**.

---

## 1. 배포 직전 체크리스트

### 1.1 코드 / 빌드

- [ ] CI 모두 green
- [ ] 코드 리뷰 통과 (2명 이상)
- [ ] release notes 작성
- [ ] Version bump (semver)

### 1.2 데이터베이스

- [ ] Flyway 마이그레이션 staging 검증 통과
- [ ] 큰 ALTER 의 expand/contract 검토
- [ ] backup 최신 (RDS snapshot)

### 1.3 보안

- [ ] secret 변경 시 graceful rotation
- [ ] CORS / CSP / SecurityConfig 변경 검토
- [ ] Audit log 항목 누락 X

### 1.4 운영

- [ ] 알람 채널 active (PagerDuty / Slack)
- [ ] 모니터링 대시보드 추가 metrics 있으면 update
- [ ] Rollback 절차 명시
- [ ] Off-peak 시간 (새벽 2-4시 한국 시간)

---

## 2. 배포 직후 (첫 30분)

```bash
# 1. Health check
curl https://api.example.com/actuator/health
# 기대: { "status": "UP" }

# 2. 가입 smoke test
curl -X POST https://api.example.com/api/v1/auth/signup \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"smoke-$(date +%s)@example.com\",\"password\":\"Tr0ub4dor!12\",\"name\":\"Smoke\",\"termsAgreed\":true}"

# 3. 로그인 smoke test
curl -X POST https://api.example.com/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d "{\"email\":\"smoke-...\",\"password\":\"Tr0ub4dor!12\"}"

# 4. /me smoke test
curl https://api.example.com/api/v1/me \
  -H "Authorization: Bearer ${ACCESS_TOKEN}"

# 5. Swagger UI 차단 검증 (prod)
curl https://api.example.com/swagger-ui/index.html
# 기대: 404 or 401

# 6. Metrics scrape
curl https://api.example.com/actuator/prometheus | grep signup
```

### 2.1 모니터링 대시보드

30분간 지속 관찰:
- [ ] 5xx 비율 < 1%
- [ ] p95 latency 정상 범위
- [ ] signup / login 성공률 정상
- [ ] DB connection pool < 70%
- [ ] JVM heap < 75%
- [ ] Email outbox 큐 길이 정상

이상 시 즉시 rollback.

---

## 3. 장애 대응 — 시나리오별 runbook

### 3.1 5xx 폭증

```
1. 어디서 ? (Grafana / Datadog)
   - 특정 endpoint? 특정 user pool?
2. 최근 배포? (Argo Rollouts / GitHub)
   - YES → rollback 검토
3. 외부 의존성? (SES / SMS / DB)
   - status page 확인
4. 자원 부족? (DB pool / JVM heap)
   - scale-out
5. 영향 범위 사용자 알림 (status page)
6. 복구 후 postmortem 작성
```

---

### 3.2 DB Connection Pool 고갈

```
증상:
  - 5xx 폭증 (특히 "Could not get connection")
  - Hikari "HikariPool-1 - Connection is not available"

대응:
  1. 메트릭 — pool active vs max
  2. Long-running query 확인 (pg_stat_activity)
     SELECT pid, query, state, query_start FROM pg_stat_activity
       WHERE state = 'active' ORDER BY query_start;
  3. 죽은 connection — kill
     SELECT pg_terminate_backend(pid) FROM pg_stat_activity
       WHERE state = 'idle in transaction' AND state_change < now() - INTERVAL '5 minutes';
  4. 일시적 scale-out (application 인스턴스 ↑)
  5. 원인 분석 — N+1? 트랜잭션 leak?

복구 후:
  - HikariCP `leak-detection-threshold` 활성
  - long query 의 timeout 정책 (statement_timeout)
```

---

### 3.3 JVM Heap 폭증 (OOM)

```
증상:
  - JVM heap > 90%
  - GC overhead ↑

대응:
  1. Heap dump (JFR / jcmd)
     jcmd $PID GC.heap_dump /tmp/heap.hprof
  2. Eclipse MAT / VisualVM 분석
  3. 일시적 — 인스턴스 재시작
  4. 원인 — N+1 / 메모리 leak

함정:
  - Heap dump 가 secret 포함 — 분석 후 즉시 폐기.
```

---

### 3.4 Email Outbox 워커 다운

```
증상:
  - email_outbox.pending.gauge > 1000
  - 사용자 "인증 메일 안 와요" CS

대응:
  1. 워커 인스턴스 status 확인
  2. 워커 로그 — exception?
  3. ShedLock 의 stale lock?
     SELECT * FROM shedlock WHERE name = 'emailOutboxWorker' AND lock_until > now();
  4. SES 의 rate limit 초과?
  5. 워커 재시작
  6. 점진적 큐 소진

함정:
  - 일괄 재시도 → SES rate limit hit → 더 큰 실패.
  - backoff 정책 활용.
```

---

### 3.5 Brute Force 의심

```
증상:
  - Login failure rate > 30%
  - 같은 IP 의 polling

대응:
  1. IP 분석 (auth_audit_log)
     SELECT ip_address, COUNT(*) FROM auth_audit_log
       WHERE event_type = 'LOGIN_FAILED' AND created_at > now() - INTERVAL '15 minutes'
       GROUP BY ip_address ORDER BY count DESC LIMIT 10;
  2. WAF 의 manual block 추가
  3. 영향받은 user 식별 — 평소와 다른 IP 시도?
  4. user 강제 logout 고려 (해당 user 만)
  5. PIPC 신고 검토 (사용자 영향 시)
```

---

### 3.6 보안 사고 (계정 탈취 / 도난)

```
1. 영향 분석
   - 어떤 user? 어떤 데이터?
   - auth_audit_log 분석

2. 즉시 mitigation
   - 영향받은 user 모두 강제 logout
     UPDATE refresh_tokens SET status = 'REVOKED',
       revoked_reason = 'SECURITY_INCIDENT'
       WHERE user_id IN (...) AND status = 'ACTIVE';
   - 비밀번호 강제 reset
     UPDATE users SET password_hash = NULL, status = 'FORCE_RESET'
       WHERE id IN (...);

3. 통보
   - 영향받은 user 알림 (이메일 + 화면)
   - 72시간 내 PIPC 신고 (개인정보보호법)
   - 사용자에게 password reset 안내

4. 사후 분석
   - 어떻게 침투?
   - 보안 강화 (rate limit 강화 / WAF rule)
   - postmortem

5. 외부 communication
   - 사용자에게 사고 공시
   - 언론 / 사용자 신뢰 복구
```

---

### 3.7 모든 사용자 강제 logout

```bash
# 보안 사고 시 — 모든 RT revoke
UPDATE refresh_tokens
SET status = 'REVOKED', revoked_at = now(), revoked_reason = 'SECURITY_INCIDENT'
WHERE status = 'ACTIVE';

# access token 은 15분 후 자연 만료
# OR 즉시 효과 = JWT secret rotation (graceful)
```

---

## 4. Postmortem Template

```markdown
# Incident YYYY-MM-DD: <짧은 제목>

## Summary
- 발생 시각:
- 복구 시각:
- 영향 사용자 수:
- 영향 데이터:

## Timeline
- 14:00 — 첫 알람
- 14:05 — on-call 인지
- 14:10 — 원인 의심
- 14:15 — rollback 시작
- 14:30 — 복구 확인

## Root Cause
(기술적 원인 + 왜 거기까지 갔는지)

## Impact
- 사용자 영향:
- 데이터 영향:
- 비즈니스 영향 (가입 / 결제 손실):

## What Went Well
- 알람 잘 작동
- 빠른 복구
- ...

## What Went Wrong
- ...

## Action Items
- [ ] (담당자, 기한)
- [ ] ...

## Lessons Learned
- ...
```

### 4.1 왜 blameless

- "누가 잘못" 이 아닌 "어떻게 막을 수 있나".
- 사람 비난 = silent (다음에 사고 숨김).
- 시스템 / 프로세스 개선 우선.

---

## 5. On-call 정책

### 5.1 Rotation

- 주 단위 — 보통 1주 on-call.
- Pager + handoff doc.
- 평일 대 주말 부담 균형.

### 5.2 응답 시간

| 우선 | 응답 시간 |
| --- | --- |
| P1 | 5분 |
| P2 | 30분 |
| P3 | 영업일 |

### 5.3 Escalation

- P1 응답 X (5분) → 다음 on-call → 매니저.

---

## 6. 정기 drill

### 6.1 Game Day

- 분기마다 — 모의 장애 시나리오.
- 실제 환경 (staging) 에서 실행.
- 팀의 runbook 이해도 검증.

### 6.2 Chaos Engineering

- prod 의 일부 서비스 의도적 down (Chaos Monkey).
- 시스템의 resilience 검증.
- 본 vault 의 규모엔 과잉 — 큰 SaaS 만.

---

## 7. 함정 모음

### 함정 1 — Runbook 없음
사고 시 우왕좌왕.
→ 시나리오별 명문화.

### 함정 2 — Runbook 이 outdated
시스템 변경 후 update 누락.
→ 정기 review + drill.

### 함정 3 — On-call rotation 없음
한 사람만 부담.
→ 팀 단위 rotation.

### 함정 4 — Smoke test 없음
배포 후 검증 X.
→ 자동 smoke test.

### 함정 5 — Postmortem 의 blame
"누가 잘못" → silent.
→ blameless culture.

### 함정 6 — Action item 의 follow-up 없음
같은 사고 반복.
→ 매주 review.

### 함정 7 — PIPC 신고 의무 누락
72시간 내 미신고 = 가중 처벌.
→ runbook 에 절차.

### 함정 8 — Rollback 절차 안 익숙
사고 시 처음 시도 = 실수.
→ drill / staging 에서 정기 연습.

### 함정 9 — 모든 알람 = P1
중요도 무력화.
→ P1/P2/P3 명확.

### 함정 10 — 알람 acknowledge 없음
같은 알람 반복.
→ acknowledge + 상태 기록.

### 함정 11 — Heap dump 분석 후 잔존
secret 포함된 dump 가 공유 저장소에.
→ 즉시 폐기.

### 함정 12 — 사용자 communication 안 함
신뢰 손상.
→ status page + 메일 / 화면 알림.

---

## 8. 관련

- [[operations|↑ operations hub]]
- [[deployment]] — 배포 전 / 후
- [[observability]] — 알람 / 메트릭
- [[../security/security]] — 보안 사고 대응
- [[../security/audit-logging]] — 사고 추적
