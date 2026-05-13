---
title: "CORS 디버깅 / 흔한 오류"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T23:30:00+09:00
tags:
  - network
  - http
  - cors
  - troubleshooting
---

# CORS 디버깅 / 흔한 오류

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 흔한 오류 + 진단 + 해결 |

**[[cors|↑ CORS]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 진단 — 단계

```
1. 브라우저 콘솔 메시지 확인 (Chrome DevTools)
2. Network 탭 — Preflight (OPTIONS) 와 본 요청 응답 헤더
3. curl 로 시뮬레이션 (preflight + 본 요청)
4. 서버 응답 헤더 검증
```

---

## 2. 흔한 오류 1 — 응답에 Allow-Origin 없음

### 콘솔 메시지

```
Access to fetch at 'https://api.example.com/users' from origin 'https://app.example.com'
has been blocked by CORS policy:
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

### 원인
- 서버가 CORS 미설정
- 잘못된 경로 (404 인데 4xx 도 CORS 헤더 X)

### 해결
- 서버에 CORS 미들웨어 추가
- 4xx / 5xx 응답도 CORS 헤더 포함하도록

---

## 3. 흔한 오류 2 — Origin 매칭 X

### 콘솔 메시지

```
The 'Access-Control-Allow-Origin' header has a value 'https://other.com'
that is not equal to the supplied origin.
```

### 원인
- Origin 명시했지만 다른 값
- 응용이 다른 origin 만 허용

### 해결
- ALLOWED_ORIGINS 에 추가
- 동적 origin 검증 + 화이트리스트

---

## 4. 흔한 오류 3 — Wildcard + Credentials

### 콘솔 메시지

```
The value of the 'Access-Control-Allow-Origin' header in the response
must not be the wildcard '*' when the request's credentials mode is 'include'.
```

### 원인
- `Allow-Origin: *` + `credentials: 'include'` (클라) + `Allow-Credentials: true` (서버)

### 해결
- 명시적 origin 사용 (동적)
- Vary: Origin 추가

---

## 5. 흔한 오류 4 — Method 거부

### 콘솔 메시지

```
Method PUT is not allowed by Access-Control-Allow-Methods in preflight response.
```

### 원인
- 응용이 `Access-Control-Allow-Methods` 에 PUT 누락

### 해결
```http
Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
```

---

## 6. 흔한 오류 5 — Header 거부

### 콘솔 메시지

```
Request header field authorization is not allowed by Access-Control-Allow-Headers in preflight response.
```

### 원인
- `Access-Control-Allow-Headers` 에 Authorization 누락
- 또는 wildcard `*` 사용 (credentials 시 적용 X)

### 해결
```http
Access-Control-Allow-Headers: Content-Type, Authorization, X-CSRF-Token
```

---

## 7. 흔한 오류 6 — Preflight 실패

### 콘솔 메시지

```
Response to preflight request doesn't pass access control check:
It does not have HTTP ok status.
```

### 원인
- OPTIONS 가 401 / 403 / 500
- 인증 미들웨어가 OPTIONS 도 차단

### 해결
- OPTIONS 인증 면제
- 또는 OPTIONS 항상 204 응답

---

## 8. 흔한 오류 7 — Redirect

### 콘솔 메시지

```
Redirect from 'https://api.example.com/user' to 'https://api.example.com/users'
has been blocked by CORS policy.
```

### 원인
- 301/302 redirect 가 CORS 안 통과
- 일부 라우터 / proxy 가 자동 redirect

### 해결
- 정확한 URL 사용
- Trailing slash 일관 (`/users` vs `/users/`)
- Redirect 응답에도 CORS 헤더

---

## 9. 흔한 오류 8 — 응답 헤더 안 보임

### 증상
JS 가 응답 헤더 (X-Total-Count 등) 못 읽음.

### 원인
- 기본 7 개 헤더 외는 명시적 expose 필요

### 해결
```http
Access-Control-Expose-Headers: X-Total-Count, X-Page-Number
```

---

## 10. 흔한 오류 9 — Cookie 안 첨부

### 증상
- 로그인은 OK
- 후속 요청에 cookie X (401)

### 원인
- 클라: `credentials: 'include'` 누락
- 서버: `Allow-Credentials: true` 누락
- Cookie: `SameSite=None` + `Secure` 누락

### 해결
```javascript
fetch(url, {credentials: 'include'});
```

```http
Set-Cookie: ...; SameSite=None; Secure
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

---

## 11. 흔한 오류 10 — 캐시 오염

### 증상
같은 URL — 어떤 사용자는 OK, 다른 사용자는 CORS error.

### 원인
- 한 origin 의 응답이 다른 origin 에 캐시됨
- Vary: Origin 누락

### 해결
```http
Vary: Origin
```

---

## 12. 디버깅 도구

### Chrome DevTools

#### Network 탭
- 빨간 X 표시 — CORS 실패
- 응답 헤더 확인
- Initiator → 코드 trace

#### Console
- CORS error 상세 메시지
- "Failed to fetch" 일반화

### curl

```bash
# Preflight 시뮬레이션
curl -X OPTIONS https://api.example.com/users \
  -H "Origin: https://app.example.com" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: Content-Type, Authorization" \
  -v

# 응답 헤더 검증
# Access-Control-Allow-Origin: ...
# Access-Control-Allow-Methods: ...
# Access-Control-Allow-Headers: ...
# Access-Control-Allow-Credentials: ...
# Access-Control-Max-Age: ...
```

### 본 요청

```bash
curl -X POST https://api.example.com/users \
  -H "Origin: https://app.example.com" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ..." \
  -d '{...}' \
  -v
```

---

## 13. 흔한 잘못 — 응용 코드

### 13.1 OPTIONS 에 미들웨어 적용
```python
# 잘못
@app.before_request
def auth():
    if not request.headers.get('Authorization'):
        return 401, "Unauthorized"
# → OPTIONS 도 401 → preflight 실패
```

### 13.2 origin 하드코딩
```python
response.headers['Access-Control-Allow-Origin'] = 'https://app.example.com'
# → 다른 환경 (staging) 안 됨. 환경별 분리.
```

### 13.3 한 응답에 CORS 헤더 두 번
- 미들웨어 중복 → 브라우저 거부

### 13.4 wildcard 와 credentials 혼합
- 사용자 입력 → `Allow-Origin: *` 실수

---

## 14. CORS 미들웨어 권장 (Production)

### 화이트리스트 + 동적

```python
ALLOWED_ORIGINS = [
    "https://app.example.com",
    "https://www.example.com",
    "https://staging.example.com",
]

@app.after_request
def cors(response):
    origin = request.headers.get('Origin')
    if origin in ALLOWED_ORIGINS:
        response.headers['Access-Control-Allow-Origin'] = origin
        response.headers['Access-Control-Allow-Credentials'] = 'true'
        response.headers['Access-Control-Allow-Methods'] = 'GET, POST, PUT, DELETE, OPTIONS'
        response.headers['Access-Control-Allow-Headers'] = 'Content-Type, Authorization'
        response.headers['Access-Control-Max-Age'] = '86400'
        response.headers['Vary'] = 'Origin'
    return response
```

### Wildcard origin 대 정규식

```python
# 위험 — 모든 subdomain 허용 가능
if origin.endswith('.example.com'):
    ...

# → subdomain takeover 시 다른 도메인이 침입 가능
# 명시 origin 권장
```

---

## 15. 빠른 점검 체크리스트

1. [ ] 서버가 `Access-Control-Allow-Origin` 보냄?
2. [ ] Origin 값이 명시 (Wildcard 아님)?
3. [ ] Preflight 가 204 / 200?
4. [ ] `Allow-Methods` 에 본 요청의 메서드 포함?
5. [ ] `Allow-Headers` 에 본 요청의 custom 헤더 포함?
6. [ ] Credentials 시 `Allow-Credentials: true` ?
7. [ ] `Vary: Origin` 있음?
8. [ ] OPTIONS 가 인증 미들웨어 면제?
9. [ ] SameSite=None + Secure (Cookie cross-origin) ?
10. [ ] Redirect 응답에도 CORS 헤더?

---

## 16. 학습 자료

- MDN CORS errors
- "Debugging CORS errors" — web.dev
- Stack Overflow CORS 태그

---

## 17. 관련

- [[cors]] — CORS hub
- [[preflight]] / [[simple-request]] / [[cors-with-credentials]]
- [[../methods/options]]
