---
title: "HTTP 압축 — gzip / brotli / zstd"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:50:00+09:00
tags:
  - network
  - http
  - performance
  - compression
  - gzip
  - brotli
---

# HTTP 압축 — gzip / brotli / zstd

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 알고리즘 비교 + 협상 |

**[[performance|↑ Performance]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

응답 body 를 **압축** 해 전송 크기 ↓. `Content-Encoding` 헤더. HTTP 의 가장 효과적
최적화 — 텍스트는 70-90% 감소.

---

## 2. 협상

### 요청
```http
Accept-Encoding: gzip, br, zstd, deflate
```

### 응답
```http
Content-Encoding: br
```

---

## 3. 알고리즘 비교

| 알고리즘 | 크기 (vs raw) | 압축 속도 | 압축 해제 | 사용 |
| --- | --- | --- | --- | --- |
| **identity** (압축 X) | 100% | — | — | 기본 |
| **deflate** | ~30-40% | 빠름 | 빠름 | 옛, 사용 X |
| **gzip** | ~30-40% | 빠름 | 빠름 | 보편 |
| **brotli (br)** | ~25-30% | 보통 | 빠름 | 모던 |
| **zstd** | ~25-35% | 빠름 | 매우 빠름 | 모던 |

### Brotli 의 장점
- gzip 보다 15-20% 더 작음
- 압축 해제 비슷
- 압축 시 비쌈 (실시간 어려움 — 정적 미리 압축)

### Zstd 의 장점
- 빠름 (압축 + 해제)
- 압축 수준 광범위 (1-22)
- Cloudflare 채택 (2023+)

---

## 4. gzip

### 가장 보편
- DEFLATE (LZ77 + Huffman) 기반
- 1990 년대 부터 표준
- 모든 브라우저 / 서버 지원

### 압축 수준
- 1 (빠름) ~ 9 (최대)
- 보통 6 (기본)

```nginx
gzip on;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript application/xml;
gzip_min_length 1024;        # 1 KB 미만은 압축 X
```

---

## 5. Brotli (RFC 7932)

### 출시
- 2015 Google
- HTTPS 만 사용 (브라우저 보안 정책)
- 사전 (dictionary) 으로 압축 더 작게

### 활성

```nginx
# ngx_brotli module 필요
brotli on;
brotli_comp_level 6;
brotli_types text/plain text/css application/json application/javascript;
brotli_static on;            # 미리 압축된 .br 파일 사용
```

### 정적 vs 동적
- **Brotli 11** (최대) — 매우 느림 — **빌드 시 미리 압축**
- **Brotli 4-6** (런타임) — 빠름, gzip 와 비슷한 크기

---

## 6. Zstd (RFC 8478)

### 출시
- 2016 Facebook
- 빠른 압축 + 좋은 비율
- Cloudflare 채택 (2023)

### 사용
- HTTP `Content-Encoding: zstd`
- 일부 CDN — 동적 압축

---

## 7. 압축 효율 측정

### text/html
```
원본: 10 KB
gzip: 2.5 KB (75% 감소)
brotli: 2.0 KB (80% 감소)
zstd: 2.3 KB
```

### text/css
```
원본: 50 KB
gzip: 8 KB
brotli: 6.5 KB
```

### text/javascript
```
원본: 300 KB
gzip: 80 KB
brotli: 65 KB
```

---

## 8. 어떤 콘텐츠를 압축?

### 압축 효과 큼 (✅ 압축)
- **text/html**, **text/css**, **application/javascript**
- **application/json**, **application/xml**
- **text/plain**, **text/markdown**
- **SVG**, **font (woff/ttf)**

### 압축 효과 작음 (❌ 압축 X)
- **image/jpeg**, **image/png**, **image/webp** — 이미 압축
- **video/mp4**, **audio/mp3** — 이미 압축
- **application/zip**, **application/gzip** — 이미 압축
- **application/pdf** — 부분적

→ 압축된 콘텐츠 재압축 = CPU 낭비.

---

## 9. 압축 시점

### 9.1 On-the-fly (런타임)
- 매 응답 시 압축
- 응답 시간 ↑, 메모리 ↑
- 짧은 응답 / 동적

### 9.2 Pre-compressed (정적)
- 빌드 시 압축 (`app.js` → `app.js.gz`, `app.js.br`)
- 응답 시 디스크에서 읽음
- CPU 절약 + 큰 압축 수준 가능

```nginx
gzip_static on;
brotli_static on;
```

### 9.3 CDN 압축
- CDN 이 origin 의 비압축 자원 압축
- Edge 캐시

---

## 10. Vary: Accept-Encoding

```http
Vary: Accept-Encoding
```

→ 캐시 별 인코딩 분리.

자세히 → [[../caching/vary-header]]

---

## 11. 함정

### 함정 1 — 이미 압축된 콘텐츠
JPEG / PNG / MP4 재압축 — 크기 변화 X + CPU 낭비.

### 함정 2 — 작은 응답 압축
< 1 KB — gzip 의 header 오버헤드 가 더 큼. min_length 설정.

### 함정 3 — BREACH / CRIME 공격
HTTPS + 압축 + 사용자 입력 — 토큰 추측 (timing).

방어:
- 비밀 정보 = 별도 응답
- Compression 끄기 (특정 endpoint)
- Token rotation

### 함정 4 — Brotli 11 의 비용
매 응답에 brotli -11 = CPU 폭증. 정적만.

### 함정 5 — gzip-bomb
악의적 작은 압축이 큰 본문 — DoS.

### 함정 6 — Accept-Encoding 무시
서버가 클라가 지원하는 인코딩 확인 안 함 → 깨진 응답.

### 함정 7 — Vary 누락
캐시가 다른 인코딩 응답 잘못 반환.

---

## 12. 측정

### 응답 크기
```bash
curl -H "Accept-Encoding: gzip" https://example.com/ -I
# Content-Length: 2500
# Content-Encoding: gzip
```

### 실제 압축률
```bash
curl -H "Accept-Encoding: gzip" -o response.gz https://...
gunzip response.gz
wc -c response                # 원본 크기
wc -c response.gz             # 압축 크기
```

### Lighthouse / WebPageTest
- 자동 압축 검증

---

## 13. 학습 자료

- **RFC 1952** (gzip), **RFC 7932** (Brotli), **RFC 8478** (Zstd)
- "Brotli vs gzip" — Cloudflare blog
- "Smaller HTML payloads with Service Workers" — web.dev

---

## 14. 관련

- [[performance]] — Performance hub
- [[transfer-encoding]] — TE vs CE
- [[../headers/entity-headers]] — Content-Encoding
- [[../headers/content-negotiation]] — Accept-Encoding
- [[../caching/vary-header]]
