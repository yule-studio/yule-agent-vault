---
title: "Referrer-Policy"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:25:00+09:00
tags:
  - network
  - http
  - security
  - referrer
  - privacy
---

# Referrer-Policy

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 8 가지 정책 / 프라이버시 |

**[[security|↑ Security]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

브라우저가 다음 요청에 **Referer 헤더 (출처 URL) 를 얼마나 보낼지** 제어. 프라이버시 + 보안.

```http
Referrer-Policy: strict-origin-when-cross-origin
```

---

## 2. 왜

### Referer 헤더의 정보
```http
Referer: https://example.com/private/secret-page?token=abc123
```

→ 다른 사이트로 navigate / 자원 요청 시:
- URL 의 query string (token, password 등)
- 페이지 경로
- 사용자 활동

→ 프라이버시 누출.

---

## 3. 8 가지 정책

| 값 | 동작 |
| --- | --- |
| **no-referrer** | Referer 절대 X |
| **no-referrer-when-downgrade** | HTTPS → HTTP 시만 X |
| **origin** | Referer 의 origin 만 (path 제거) |
| **origin-when-cross-origin** | same-origin = full, cross-origin = origin |
| **same-origin** | same-origin 만 보냄 |
| **strict-origin** | HTTPS → HTTPS 만, origin 만 |
| **strict-origin-when-cross-origin** ⭐ | same = full, cross HTTPS = origin, HTTPS→HTTP = X |
| **unsafe-url** | 항상 full URL (위험) |

### 브라우저 기본 (모던)
- Chrome 85+ / Firefox 87+ / Safari 모던: **strict-origin-when-cross-origin**
- 옛: no-referrer-when-downgrade

---

## 4. 값별 동작 예

```
페이지: https://example.com/private/page?token=abc
다음 요청: https://api.com/...
```

| 정책 | Referer 보냄 |
| --- | --- |
| no-referrer | (없음) |
| no-referrer-when-downgrade | `https://example.com/private/page?token=abc` (HTTPS→HTTPS 라 보냄) |
| origin | `https://example.com/` |
| origin-when-cross-origin | `https://example.com/` (cross-origin) |
| same-origin | (없음 — cross-origin 이므로) |
| strict-origin | `https://example.com/` (HTTPS→HTTPS) |
| **strict-origin-when-cross-origin** | `https://example.com/` |
| unsafe-url | `https://example.com/private/page?token=abc` |

### Same-origin 경우
```
다음 요청도 https://example.com/...
```

→ `strict-origin-when-cross-origin` 은 full URL 보냄.

---

## 5. 설정 방법

### HTTP 헤더 (응답)
```http
Referrer-Policy: strict-origin-when-cross-origin
```

### HTML meta
```html
<meta name="referrer" content="strict-origin-when-cross-origin">
```

### 개별 link / form
```html
<a href="..." referrerpolicy="no-referrer">link</a>
<form action="..." referrerpolicy="origin">...</form>
```

### iframe
```html
<iframe src="..." referrerpolicy="no-referrer"></iframe>
```

---

## 6. 권장

### 일반 사이트
```http
Referrer-Policy: strict-origin-when-cross-origin
```

→ 브라우저 기본. Cross-origin 에는 path 제거.

### 민감 페이지 (결제 / 인증 후)
```http
Referrer-Policy: no-referrer
```

→ 어디 가든 path 노출 X.

### 광고 / 분석
```http
Referrer-Policy: origin-when-cross-origin
```

→ origin 만 알림 — 분석 OK + path 보호.

---

## 7. 보안 / 프라이버시 효과

### 7.1 Query string 의 비밀 보호
```
URL: /reset-password?token=abc123
→ 다른 사이트로 이동 시 token 누출
→ Referrer-Policy 로 차단
```

### 7.2 사용자 추적 방지
```
Referer 로 사용자 행동 추적 (어떤 페이지에서 왔는지)
→ 광고 / 분석의 fingerprinting
```

### 7.3 내부 URL 보호
```
/admin/dashboard
/internal/api/...
→ 외부 자원 요청 시 노출 X
```

---

## 8. 서버 설정

### Nginx
```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

### Apache
```apache
Header always set Referrer-Policy "strict-origin-when-cross-origin"
```

### Express
```javascript
app.use((req, res, next) => {
    res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
    next();
});

// 또는 helmet
const helmet = require('helmet');
app.use(helmet.referrerPolicy({policy: 'strict-origin-when-cross-origin'}));
```

---

## 9. CSRF 방어와의 관계

CSRF 의 Referer 검증:
```python
def check_csrf(request):
    referer = request.headers.get('Referer')
    if not referer or not referer.startswith('https://app.example.com'):
        return 403
```

→ Referrer-Policy 가 너무 엄격 (no-referrer) 시 깨짐.

### 해결
- `strict-origin-when-cross-origin` — 충분 (origin 매칭)
- 또는 SameSite Cookie + CSRF Token 으로 대체

---

## 10. 함정

### 함정 1 — no-referrer 의 영향
- 분석 깨짐
- CSRF Referer 검증 깨짐
- 페이지 별 trade-off

### 함정 2 — unsafe-url
- URL 의 모든 비밀 노출
- 거의 X

### 함정 3 — Referer 신뢰
- 사용자 수정 가능 (curl, browser extension)
- 인증 / 인가 신뢰 X — 분석 만

### 함정 4 — 옛 브라우저
- IE 11 — 일부 지원 X
- 폴백: no-referrer-when-downgrade

### 함정 5 — meta vs 헤더
- 둘 다면 더 엄격 적용
- 일관 유지

---

## 11. 학습 자료

- "Referrer-Policy" — W3C
- MDN Referrer-Policy
- "A new default Referrer-Policy" — Google blog (2020 변경)

---

## 12. 관련

- [[security]] — Security hub
- [[../headers/request-headers]] — Referer / Origin
- [[../cors/cors]] — Origin
