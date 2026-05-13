---
title: "Session Cookies & 서버 세션"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T22:55:00+09:00
tags:
  - network
  - http
  - cookies
  - session
---

# Session Cookies & 서버 세션

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | Session ID / Store / Stateful vs Stateless |

**[[cookies|↑ Cookies]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

**HTTP stateless** 한계 보완 — 서버가 세션 상태를 메모리/DB 에 저장, 쿠키엔 Session ID 만.

---

## 2. 전통적 흐름

```
1. 로그인 (POST /login)
2. 서버: 사용자 인증 → 세션 생성
   - Session ID = random UUID
   - Session Store 에 저장 (user_id, expires, ...)
3. 응답: Set-Cookie: sessionid=<random>
4. 후속 요청: 자동 Cookie: sessionid=<random>
5. 서버: Session Store 에서 user_id 조회
6. 인증된 사용자로 처리
```

---

## 3. Session Store 종류

### 3.1 메모리 (단일 서버)

```python
# 단순 — 단일 인스턴스만
sessions = {}    # in-memory dict

def login(user, password):
    sid = str(uuid.uuid4())
    sessions[sid] = {"user_id": user.id, "expires": time.time() + 3600}
    return sid
```

### 한계
- 서버 재시작 시 모두 X
- 여러 인스턴스 → 세션 공유 X
- 메모리 사용량

### 3.2 Redis (분산 표준)

```python
import redis
r = redis.Redis()

def login(user, password):
    sid = str(uuid.uuid4())
    r.setex(f"session:{sid}", 3600, json.dumps({"user_id": user.id}))
    return sid

def get_session(sid):
    data = r.get(f"session:{sid}")
    return json.loads(data) if data else None
```

### 장점
- 여러 인스턴스 공유
- 빠름 (1 ms)
- TTL 자동 expire
- 표준 패턴

### 3.3 DB (PostgreSQL / MySQL)

```sql
CREATE TABLE sessions (
    id VARCHAR(64) PRIMARY KEY,
    user_id INT,
    data JSON,
    expires_at TIMESTAMP
);
```

### 장점
- 영속 (Redis 죽어도 OK)
- 트랜잭션 일관

### 단점
- 매 요청 DB 조회 — 느림
- Redis 캐시 권장

### 3.4 Sticky Session (Session Affinity)

- LB 가 한 사용자를 같은 인스턴스에 보냄
- 메모리 세션 가능 (인스턴스 별)
- 한계: 인스턴스 죽으면 세션 X, scale 어려움

---

## 4. Stateful vs Stateless

### Stateful — 전통 세션
- 서버가 세션 상태 보유
- Cookie 엔 ID 만
- 즉시 무효화 가능
- Scaling 시 store 공유 필요

### Stateless — JWT 등
- 서버가 상태 없음
- Token 자체에 정보
- Scaling 쉬움
- 즉시 무효화 어려움 (만료까지)

자세히 → [[jwt-vs-cookie]]

---

## 5. 세션 만료 정책

### 5.1 Absolute Timeout

```
세션 생성 → 만료 시간 고정 (24 시간)
사용 무관 — 만료
```

→ 도난 시 영향 한계.

### 5.2 Idle Timeout

```
세션 사용할 때마다 만료 시간 갱신
30 분 idle → 자동 로그아웃
```

→ 사용자 경험 OK, 짧은 idle.

### 5.3 결합

```
Idle: 30 분
Absolute: 8 시간

→ 30 분 idle 또는 8 시간 절대 만료
```

→ 모범 — 은행 / 의료 등.

### 5.4 Refresh Token 패턴

```
Access Token: 짧음 (15 분)
Refresh Token: 김 (7-30 일)

Access 만료 → Refresh 로 갱신
Refresh 도난 → Refresh rotation
```

자세히 → [[../auth/auth]]

---

## 6. 로그아웃 시 처리

```python
def logout(request):
    sid = request.cookies.get('sessionid')
    if sid:
        redis.delete(f"session:{sid}")    # 서버 세션 무효
    
    response = redirect('/login')
    response.set_cookie('sessionid', '', max_age=0)   # 클라 쿠키 삭제
    return response
```

→ 서버 + 클라 둘 다 정리.

### "Logout from all devices"
- 사용자의 모든 세션 조회 → 삭제
- 또는 user 의 `tokens_invalid_before` timestamp 갱신

---

## 7. Session vs Token 표준 패턴

### Server-side rendering (전통 웹)
- Session cookie + Server-side store
- Django, Rails, Spring

### SPA + REST API
- JWT Bearer token
- Authorization 헤더
- localStorage / sessionStorage

### SPA + Cookie auth (현대 권장)
- HttpOnly + SameSite cookie
- CSRF token / SameSite=Strict
- Auth0, Supabase 패턴

### Mobile app
- JWT 또는 OAuth
- Keychain / KeyStore

---

## 8. Session Hijacking 의 cookie 보안

자세히 → [[cookie-security]]

### 핵심
- HTTPS + Secure
- HttpOnly
- SameSite=Strict
- 짧은 만료 + Refresh

---

## 9. Framework

### Express (Node.js)

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;

app.use(session({
  store: new RedisStore({client: redisClient}),
  secret: 'random-strong-secret',
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,
    httpOnly: true,
    sameSite: 'strict',
    maxAge: 3600 * 1000
  }
}));

// 사용
req.session.user_id = user.id;
req.session.destroy();
```

### Django

```python
# settings.py
SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
SESSION_CACHE_ALIAS = 'redis'
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = 'Strict'
SESSION_COOKIE_AGE = 3600

# 뷰
def login_view(request):
    ...
    request.session['user_id'] = user.id
```

### Rails

```ruby
Rails.application.config.session_store :redis_store,
  servers: 'redis://localhost:6379/0',
  expire_after: 1.hour,
  key: '_session_id',
  same_site: :strict,
  secure: true
```

### Spring Boot

```yaml
spring:
  session:
    store-type: redis
    timeout: 3600s
server:
  servlet:
    session:
      cookie:
        secure: true
        http-only: true
        same-site: strict
```

---

## 10. 함정

### 함정 1 — 메모리 세션의 scaling
LB 가 여러 인스턴스 분산 → 세션 X. Sticky Session 또는 Redis.

### 함정 2 — Session ID 의 약한 random
guess 가능. UUID v4 (122 bit random) 또는 secrets.token_urlsafe.

### 함정 3 — 무한 세션
만료 없음 → 도난 영향 영원. Absolute timeout.

### 함정 4 — Session Fixation
로그인 시 새 ID 발급 안 함. `session.regenerate()`.

### 함정 5 — Logout 시 서버 세션 유지
클라 쿠키만 삭제 → 토큰 재제출 시 살아남. 서버 store 에서도 삭제.

### 함정 6 — Session store 의 성능
DB 만 — 매 요청 부하. Redis 캐시.

### 함정 7 — 큰 세션 데이터
큰 객체 — Redis 부담. ID 만, 데이터는 DB.

### 함정 8 — Cookie 의 Domain 너무 넓음
Subdomain 침해 시 영향. `__Host-` 권장.

---

## 11. 학습 자료

- **OWASP Session Management Cheat Sheet**
- "Stateful vs Stateless authentication" 비교 가이드
- Django / Rails / Express Session 공식 문서

---

## 12. 관련

- [[cookies]] — Cookies hub
- [[cookie-attributes]]
- [[cookie-security]]
- [[jwt-vs-cookie]] — JWT 비교
- [[../auth/auth]] — 인증
