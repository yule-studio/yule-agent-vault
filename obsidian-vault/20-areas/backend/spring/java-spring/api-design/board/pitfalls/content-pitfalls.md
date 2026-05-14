---
title: "Content 함정 — XSS / markdown / 첨부"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T15:24:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - board
  - pitfalls
  - content
---

# Content 함정 — XSS / markdown / 첨부

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-15 | engineering-agent/tech-lead | 최초 |

**[[pitfalls|↑ hub]]**

---

## 함정 1 — HTML 직접 저장
XSS 가드 부담 ↑.
→ Markdown + sanitize.

## 함정 2 — 저장 시 sanitize
원본 손실.
→ raw 저장, 렌더링 시 sanitize.

## 함정 3 — Blacklist sanitize
새 element 무방비.
→ OWASP Whitelist.

## 함정 4 — `javascript:` URL 허용
XSS.
→ https / mailto only.

## 함정 5 — CSP 없음
XSS 발생 시 mitigation 없음.
→ `default-src 'self'`.

## 함정 6 — `rel="nofollow"` 누락
spam link SEO 망침.

## 함정 7 — Bean Validation 만 (도메인 검증 X)
도메인 VO 의 검증 누락.
→ 4계층 (Bean / VO / DB CHECK / DB UNIQUE).

## 함정 8 — title / content 길이 무제한
abuse — 1MB 글.
→ CHECK 200 / 50000.

## 함정 9 — 첨부 size / type 검증 X
.exe / 1GB file.
→ presigned 시 검증.

## 함정 10 — Presigned TTL 영구
도난 시 무한.
→ 5분.

## 함정 11 — S3 exists 검증 X
fake key 매핑.
→ HEAD request.

## 함정 12 — Orphan cleanup 없음
업로드만 한 file 영구.
→ S3 lifecycle.

## 함정 13 — file_name path traversal
`../../etc/passwd`.
→ sanitize + ulid prefix.

## 함정 14 — 외부 image URL 허용
viewer tracking + 외부 server 부담.
→ CDN URL 만.

## 함정 15 — 렌더링 결과 cache 없음
매 조회 markdown parse → CPU.
→ Redis cache.

---

## 관련

- [[../design-decisions/content-format]] · [[../design-decisions/attachment-storage]]
- [[../security/xss-defense]]
- [[pitfalls|↑ hub]]
