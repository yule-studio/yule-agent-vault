---
title: "auth §13 — 토큰 갱신 (refresh rotation + reuse detection)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T00:35:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - auth
  - jwt
  - refresh-token
---

# auth §13 — 토큰 갱신 (refresh rotation + reuse detection)

**[[signup|↑ hub]]**  ·  ← [[login-impl]]  ·  → [[password-reset-impl]]

> [[login-impl]] 의 RefreshTokenService 가 핵심. 본 노트는 **갱신 endpoint + rotation 정책 + reuse 감지 시나리오** 에 집중.

---

## 1. API spec

```http
POST /api/v1/auth/token/refresh
Content-Type: application/json

{ "refreshToken": "9f4b3a2c-...-base64url" }
```

```http
200 OK
{
  "code": "OK_001",
  "result": {
    "accessToken":  "eyJ...",
    "refreshToken": "newrandom...-base64url",        ← 매번 새로 (rotation)
    "tokenType":    "Bearer",
    "expiresIn":    900
  }
}
```

---

## 2. Rotation — 동작 방식

```
[클라가 보유한 RT_v1] ──→ POST /token/refresh
                                ↓
                  서버: hash 로 lookup
                  current.status == ACTIVE ?
                                ↓ Yes
                  새 RT_v2 생성 + DB INSERT (ACTIVE)
                  current.status = ROTATED
                  current.rotatedToId = RT_v2.id
                                ↓
                          { access, RT_v2 raw } 응답
                                ↓
[클라가 보유한 RT_v2 만 유효]    [DB 의 RT_v1 은 ROTATED]
```

---

## 3. Reuse Detection — 도난 시나리오

```
공격자가 RT_v1 탈취 (XSS / 네트워크 스니핑 / 디스크 dump)
       ↓
정상 user 가 RT_v1 사용 → RT_v2 발급 (이때 RT_v1 → ROTATED)
       ↓
공격자가 탈취한 RT_v1 또 사용 시도
       ↓
서버: RT_v1 hash lookup → status == ROTATED 발견
       ↓
🚨 "reuse 감지" → user 의 모든 RT REVOKE
       ↓
정상 user 의 RT_v2 도 죽음 → 강제 재로그인 (대신 안전)
       ↓
사용자에 알림 (이메일 / 푸시): "새 위치에서 로그인 시도 감지"
```

→ [[login-impl#6]] 의 `RefreshTokenService.rotate(...)` 가 이 로직 구현.

---

## 4. UseCase / Controller — 갱신 endpoint

```java
// presentation/api/v1/auth/TokenController.java
@Tag(name = "토큰")
@RestController
@RequestMapping("/api/v1/auth/token")
@RequiredArgsConstructor
public class TokenController {

    private final RefreshTokenService rtService;
    private final JwtTokenProvider jwt;
    private final UserRepository users;

    @Operation(summary = "토큰 갱신 (refresh rotation)")
    @SecurityRequirements()
    @PostMapping("/refresh")
    public ResponseEntity<CommonResponse<TokenResponse>> refresh(
        @Valid @RequestBody RefreshRequest req,
        HttpServletRequest http
    ) {
        var device = http.getHeader("User-Agent");
        var ip = ClientIpUtil.resolveClientIp(http);

        try {
            var newRt = rtService.rotate(req.refreshToken(), device, ip);

            // 새 access token — sessionId 같이 새로 발급
            var user = users.findById(newRt.userId())
                .orElseThrow(() -> new BusinessException(ResponseCode.USER_NOT_FOUND));
            var sessionId = UUID.randomUUID().toString();
            var access = jwt.generateAccessToken(user.email().value(), user.id().value(),
                                                 user.role(), sessionId);

            var body = CommonResponse.success(ResponseCode.OK,
                new TokenResponse(access, newRt.raw(), "Bearer", 900L),
                "토큰 갱신 완료"
            );
            return ResponseEntity.ok(body);

        } catch (RefreshTokenReuseDetectedException e) {
            // user 의 모든 RT REVOKE 후 알림 발행 (도메인 이벤트로)
            log.warn("RT reuse detected for user {}", e.userId());
            throw e;
        }
    }
}
```

---

## 5. 만료된 RT cleanup — Scheduler

```java
@Component
@RequiredArgsConstructor
public class RefreshTokenCleanupJob {

    private final RefreshTokenJpaRepository tokens;

    @Scheduled(cron = "0 0 4 * * *")              // 매일 새벽 4시
    @SchedulerLock(name = "refreshTokenCleanup", lockAtMostFor = "PT1H")
    public void cleanup() {
        // 만료 후 7일 지난 row 삭제 (audit 용 7일 보존)
        var cutoff = Instant.now().minus(Duration.ofDays(7));
        int deleted = tokens.deleteByExpiresAtBefore(cutoff);
        log.info("refresh token cleanup: {} rows deleted", deleted);
    }
}
```

---

## 6. 보안 / Edge case

### 6.1 stolen RT — 동시 reuse

탈취당한 후 attacker 가 RT_v1 사용 + 정상 user 가 RT_v1 사용 = race. 둘 중 1명이 rotate 성공 (status → ROTATED). 나머지는 reuse detection 발동.

→ **누가 attacker 인지 식별 불가** — 모든 RT revoke 가 정답 (안전 우선).

### 6.2 클라가 RT 갱신 후 응답 못 받음 (네트워크 실패)

```
클라: POST /refresh { RT_v1 }
서버: RT_v1 → ROTATED, RT_v2 발급
클라: 응답 받기 전에 timeout
클라: RT_v1 retry → 서버: ROTATED 감지 → "reuse detected" → 모든 RT 죽임
```

→ 정상 retry 가 reuse 로 오인. **해결 옵션**:
- (A) 응답 받을 때까지 클라가 RT_v1 보유. retry 정책. (단순, 가끔 사고)
- (B) `Idempotency-Key` 헤더 — 같은 키면 같은 응답 (RT_v2) 반환. (안전)

본 vault: **(B) 권장** — 클라가 UUID 생성 + 24h 응답 캐싱.

### 6.3 다중 디바이스 — 다른 device 갱신

각 디바이스가 자기 RT 보유 — 독립 rotation. 한 device 의 reuse 감지가 다른 device 영향 X (단, 같은 user 의 모든 RT revoke 가 영향).

→ **trade-off**: reuse 시 모든 device 강제 로그아웃 (안전) vs 해당 device 만 (UX). 본 vault: **모든 device** (안전 우선).

---

## 7. 함정 모음

### 함정 1 — RT 를 raw 저장
DB 유출 = 모든 RT 사용 가능. **SHA-256 hash**.

### 함정 2 — Rotation 없음
탈취된 RT 영구 사용 가능.

### 함정 3 — Reuse detection 후 알림 안 함
정상 user 가 강제 로그아웃됐는데 이유 모름. **이메일 / 푸시 알림** 필수.

### 함정 4 — Idempotency 없음
네트워크 retry 가 reuse 로 오인. `Idempotency-Key` 권장.

### 함정 5 — Refresh 가 access 와 같은 secret
보안 부담. 같은 HS256 secret OK 지만 type claim (`token_type`) 검증 필수.

### 함정 6 — `isRefresh(token)` 체크 누락
access token 으로 refresh endpoint 호출 시 통과. **type claim 명시 검증**.

### 함정 7 — RT 만료 cleanup 없음
DB 무한 증가. 매일 cleanup job.

### 함정 8 — Logout 후 access 도 즉시 무효 기대
JWT stateless. access 는 만료까지 살아있음. 짧은 TTL 로 감수.

---

## 8. 운영 체크리스트

- [ ] RT `token_hash` UNIQUE 인덱스
- [ ] Rotation 동작 통합 테스트
- [ ] Reuse detection 시나리오 IT (시뮬레이션)
- [ ] 만료 cleanup 매일 job (`ShedLock`)
- [ ] Reuse 감지 → 사용자 알림 (이메일 / 푸시)
- [ ] `JwtTokenProvider.isRefresh(...)` 검증 명시
- [ ] `Idempotency-Key` 지원 (권장)
- [ ] 만료된 RT 사용 시 401 + 명확한 메시지

---

## 9. 관련

- [[signup|↑ hub]]
- [[login-impl]] — 이전 (§12) — `RefreshTokenService` 정의
- [[password-reset-impl]] — 다음 (§14)
- [[../../common/security-config#7 JwtTokenProvider]]
