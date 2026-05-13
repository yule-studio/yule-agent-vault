---
title: "X-Content-Type-Options — MIME Sniffing 방어"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:20:00+09:00
tags:
  - network
  - http
  - security
  - mime-sniffing
---

# X-Content-Type-Options — MIME Sniffing 방어

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | nosniff / 보안 효과 |

**[[security|↑ Security]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

브라우저가 응답의 **MIME 타입을 추측 (sniff) 하지 못하게** 강제. `nosniff` 1 값만.

```http
X-Content-Type-Options: nosniff
```

---

## 2. MIME Sniffing 이란

### 동작
- 브라우저가 `Content-Type` 무시하고 **본문 내용으로 추측**:
  - HTML 같으면 → `text/html`
  - JS 같으면 → `application/javascript`
  - 이미지 같으면 → `image/jpeg`

### 옛 목적
- 1990 후반 — 잘못 설정된 서버 대응
- 사용자 경험 ↑

---

## 3. 위험

### 시나리오 1 — 사용자 업로드 이미지

```
사용자가 /uploads/photo.jpg 업로드
서버: Content-Type: image/jpeg
파일 내용: 사실 HTML + <script>

브라우저: "JPEG 같지 않은데 — HTML 로 해석"
→ JS 실행 (XSS)
```

### 시나리오 2 — 응답 sniffing

```
서버: Content-Type: text/plain (JSON 의도)
응답: {"data": "<script>alert(1)</script>"}

옛 IE: "HTML 같음" → 스크립트 실행
```

### 시나리오 3 — Download 의 sniff

```
사용자 다운로드 (Content-Type: application/octet-stream)
브라우저: "PDF 같음" → 직접 열기
```

→ 신뢰 안 된 콘텐츠를 브라우저가 잘못 해석 → XSS.

---

## 4. nosniff 의 효과

```http
HTTP/1.1 200 OK
Content-Type: text/plain
X-Content-Type-Options: nosniff

<script>...</script>
```

→ 브라우저: "text/plain 이라고 명시 — 절대 HTML/JS 로 해석 X". 안전.

### 영향
- 응답이 명시 Content-Type 그대로 처리
- 잘못된 Content-Type → 표시 깨짐 (디버깅 용이)

---

## 5. MIME types 의 보안 분류 (Fetch standard)

브라우저가 응답을 어떻게 처리할지 — Content-Type 별:

### Script (실행)
- `application/javascript`, `text/javascript`
- nosniff 면 정확한 type 만 script 로

### Style (CSS)
- `text/css`
- nosniff 면 다른 type 의 CSS 로딩 차단

→ 잘못된 type 의 자원 — 차단 또는 무시.

---

## 6. 권장 — 모든 응답

```http
X-Content-Type-Options: nosniff
```

→ 보편 권장. 거의 사고 X.

---

## 7. 서버 설정

### Nginx
```nginx
add_header X-Content-Type-Options "nosniff" always;
```

### Apache
```apache
Header always set X-Content-Type-Options "nosniff"
```

### Express
```javascript
app.use((req, res, next) => {
    res.setHeader('X-Content-Type-Options', 'nosniff');
    next();
});

// 또는 helmet 자동
const helmet = require('helmet');
app.use(helmet());
```

### Cloudflare
- "Always Use HTTPS" + 다른 보안 헤더와 함께
- Edge Rules 로 자동

---

## 8. 파일 업로드 시 추가 방어

nosniff 만으로는 부족한 경우:

### 8.1 Content-Disposition: attachment

```http
Content-Type: application/octet-stream
Content-Disposition: attachment; filename="user-upload.jpg"
X-Content-Type-Options: nosniff
```

→ 브라우저가 표시 X — 다운로드 강제. 절대 안전.

### 8.2 별도 도메인 / 서브도메인

```
앱: app.example.com
업로드: uploads.example.com (또는 별도 도메인)
```

→ XSS 가 일어나도 다른 origin — 영향 적음.

### 8.3 콘텐츠 검증

- 업로드 시 magic bytes 검증
- ImageMagick / sharp 으로 재인코딩
- 사용자 입력 sanitize

자세히 → [[../methods/post]] 의 multipart

---

## 9. 함정

### 함정 1 — 잘못된 Content-Type
서버가 자동 추측 / 잘못 설정 → nosniff 면 표시 깨짐. 명시 설정.

### 함정 2 — 모든 응답에 동일
HTML 페이지에도 `application/octet-stream` 보내면 표시 X. type 별 맞춤.

### 함정 3 — 옛 브라우저
- IE 9 미만 — nosniff 무시
- 모던 브라우저 — 표준 지원

### 함정 4 — Content-Type 누락
- 누락 시 nosniff 의미 모호
- 기본 동작 브라우저 별 — 항상 명시.

### 함정 5 — Style 의 type 검증
모던 브라우저: `<link rel="stylesheet">` 의 응답이 `text/css` 아니면 무시 (nosniff 시).

---

## 10. 학습 자료

- "X-Content-Type-Options" — MDN
- "MIME sniffing" — Fetch standard
- OWASP "Securing Web Headers"

---

## 11. 관련

- [[security]] — Security hub
- [[../headers/entity-headers]] — Content-Type
- [[../methods/post]] — 파일 업로드
