---
title: "3xx — Redirection"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T21:20:00+09:00
tags:
  - network
  - http
  - status-codes
  - 3xx
  - redirect
---

# 3xx — Redirection

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 300/301/302/303/304/307/308 |

**[[status-codes|↑ Status Codes]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

"추가 동작 필요" — 보통 다른 URI 로 이동 (Location 헤더). 304 는 캐시 사용.

---

## 2. 표준 코드

### 300 Multiple Choices

```http
HTTP/1.1 300 Multiple Choices
Content-Type: text/html

<ul>
  <li><a href="/page.en">English</a></li>
  <li><a href="/page.ko">Korean</a></li>
</ul>
```

여러 표현 — 클라가 선택. 거의 사용 X — Content Negotiation 이 자동 처리.

### 301 Moved Permanently

```http
GET /old HTTP/1.1

HTTP/1.1 301 Moved Permanently
Location: /new
```

#### 동작
- **영구 이전** — 검색엔진이 색인 갱신
- 브라우저 / 검색엔진이 캐시 (다음부터 직접)
- HTTP → HTTPS 강제, www ↔ apex

#### 메서드 처리 (역사적 모호)
- 옛 RFC: 메서드 변경 가능 (POST → GET)
- 실제: 모든 메서드 보존 권장
- 명확히는 **308** 사용

### 302 Found

```http
HTTP/1.1 302 Found
Location: /temp
```

#### 동작
- **임시 이전** — 캐시 X
- 검색엔진 색인 유지

#### 메서드 처리 (역사적 모호)
- 옛 의도: POST → GET (현실 동작)
- RFC: 메서드 보존
- → 실제로 POST 가 GET 으로 변경됨 (사실상 303)

#### 사용
- "역사적 잘못된 디자인" — 모던은 303 / 307 명확히

### 303 See Other

```http
POST /submit HTTP/1.1

HTTP/1.1 303 See Other
Location: /thanks
```

#### 동작
- **반드시 GET 으로 변경** + Location 으로 GET
- "처리는 끝났어, 결과는 이 URL 에"

#### PRG 패턴 (Post-Redirect-Get)
- 폼 제출 후 새로고침 시 재제출 방지
- POST /submit → 303 → GET /thanks
- 사용자 새로고침 = GET /thanks 반복 (안전)

### 304 Not Modified

```http
GET /api/users/123 HTTP/1.1
If-None-Match: "v1"

HTTP/1.1 304 Not Modified
ETag: "v1"
Cache-Control: max-age=300
(body 없음)
```

#### 동작
- 캐시 valid — body 보내지 X
- 조건부 GET 의 응답
- body 없음 (`Content-Length: 0` 또는 생략)

자세히 → [[../caching/etag-conditional]]

#### 헤더
- ETag / Last-Modified 갱신 OK
- Cache-Control / Date / Vary 같이

### 305 Use Proxy (deprecated)
보안상 폐기.

### 306 (Reserved)
사용 X.

### 307 Temporary Redirect

```http
POST /submit HTTP/1.1

HTTP/1.1 307 Temporary Redirect
Location: /backup-server/submit
```

#### 동작
- **임시 이전 + 메서드 보존**
- POST → POST (302 의 명확화)

#### 사용
- 임시 서버 이동
- A/B test 의 일부
- Maintenance 우회

### 308 Permanent Redirect (RFC 7538)

```http
POST /old HTTP/1.1

HTTP/1.1 308 Permanent Redirect
Location: /new
```

#### 동작
- **영구 이전 + 메서드 보존**
- 301 의 명확화

#### 사용
- HTTPS 강제 (301 이지만 308 도 가능)
- API 영구 이전
- 모던 권장

---

## 3. 301 / 302 / 303 / 307 / 308 비교

| 코드 | 영구 / 임시 | 메서드 보존 | 캐시 |
| --- | --- | --- | --- |
| **301** | 영구 | 모호 (보통 보존) | ✅ |
| **302** | 임시 | 모호 (보통 GET 으로) | ❌ |
| **303** | 임시 | **GET 으로 강제** | ❌ |
| **307** | 임시 | **보존** | ❌ |
| **308** | 영구 | **보존** | ✅ |

### 결정 트리

```
영구 / 임시?
├ 영구
│   ├ 메서드 보존? → 308
│   └ 모호 OK → 301 (보통)
└ 임시
    ├ POST → GET (PRG)? → 303
    ├ 메서드 보존? → 307
    └ 모호 OK → 302
```

---

## 4. Location 헤더

```http
Location: /new-path                       (상대)
Location: https://other.example.com/path  (절대)
Location: ./relative                      (현재 기준)
```

- 모든 3xx (304 제외) 에 필수
- 절대 / 상대 모두 OK
- 모던: 절대 권장

---

## 5. Redirect Loop

```
/a → 302 → /b
/b → 302 → /a
→ 무한 루프
```

### 방어
- 브라우저: 5-20 회 후 중단
- curl: `-L --max-redirs 10` (기본 50)
- requests (Python): `max_redirects=30`

### 디버깅
```bash
curl -v -L https://example.com/old
# 모든 redirect chain 추적
```

---

## 6. 301 의 검색엔진 영향

```
구글 / 빙: 301 → PageRank 이전 (몇 주 ~ 몇 달 후)
색인 갱신:
  - 옛 URL 점진 제거
  - 새 URL 색인
```

302 / 303 / 307 — 색인 갱신 X (임시).

---

## 7. HTTPS Redirect 패턴

```nginx
# Nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

```apache
# Apache
RewriteEngine On
RewriteCond %{HTTPS} !=on
RewriteRule ^/(.*) https://example.com/$1 [R=301,L]
```

→ HSTS (Strict-Transport-Security) 도 함께 권장:
```http
HTTP/1.1 200 OK
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

자세히 → [[../security/hsts]]

---

## 8. www ↔ apex Redirect

### Apex (root) → www
```
example.com → www.example.com (301)
```

### www → Apex
```
www.example.com → example.com (301)
```

→ 둘 중 하나만 선택. SEO 정합성.

---

## 9. Path / Trailing Slash

```
/users (no slash) ↔ /users/ (with slash)
```

→ 둘 중 하나로 통일 (301). SEO 중복 방지.

- Django: `APPEND_SLASH = True` (자동 redirect)
- Express: `strict routing` 옵션
- Nginx: `rewrite ^/(.+)/$ /$1 permanent;`

---

## 10. 인증 후 Redirect

```
1. /private → 401 또는 302 → /login?next=/private
2. /login (사용자 입력)
3. POST /login → 303 → /private
4. /private → 200 OK
```

### 보안 — Open Redirect
```
/login?next=https://evil.com
```

→ 검증 필요. 자기 도메인만 허용.

자세히 → [[../security/security]]

---

## 11. 함정

### 함정 1 — 302 의 메서드 변경
POST → GET 으로 (역사적). 의도 시 **303**, 보존 시 **307**.

### 함정 2 — 301 의 강한 캐시
브라우저가 매우 오래 캐시. 잘못 설정 시 복구 어려움 (사용자 캐시 clear 필요).

### 함정 3 — Redirect Loop
방어 코드 없으면 브라우저 timeout.

### 함정 4 — Open Redirect 취약점
`?next=` 검증 X — phishing 도구화.

### 함정 5 — 304 의 헤더 누락
ETag / Cache-Control 다시 보내 — 클라 캐시 갱신.

### 함정 6 — Redirect 와 Method override
일부 라이브러리가 redirect 시 Authorization 헤더 제거 (보안). API 호출 깨짐.

### 함정 7 — Apache mod_rewrite 의 R 플래그
`[R]` 기본 302. 영구는 `[R=301]`.

### 함정 8 — Redirect with Body
RFC: 3xx 도 body OK (HTML 등). 브라우저는 보통 안 보여줌.

---

## 12. 학습 자료

- **RFC 9110** Section 15.4 (3xx)
- **RFC 7538** (308)
- MDN HTTP redirections
- Moz "301 vs 302" SEO

---

## 13. 관련

- [[status-codes]] — hub
- [[2xx-success]] — 200 OK 와 비교
- [[4xx-client-errors]] — 4xx 도 동일 폴더
- [[../caching/etag-conditional]] — 304
- [[../security/security]] — Open Redirect
- [[../headers/response-headers]] — Location
