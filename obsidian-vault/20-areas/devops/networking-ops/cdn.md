---
title: "CDN — CloudFront / Cloudflare / Fastly"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:50:00+09:00
tags: [devops, networking-ops, cdn]
---

# CDN — CloudFront / Cloudflare / Fastly

**[[networking-ops|↑ networking-ops]]**

---

## 1. 왜

- **edge POP** (전 세계 200+ 위치) 에 cache.
- 사용자 가까이에서 응답 → latency ↓.
- origin 부담 ↓.
- DDoS 흡수 (anycast network).

→ static heavy / global / 트래픽 spike 보호.

---

## 2. 도구

| | 강점 | 약점 |
| --- | --- | --- |
| **CloudFront** | AWS 통합, ACM 무료 cert | 설정 복잡 |
| **Cloudflare** | 무료 tier 강력, WAF / DDoS 우수 | enterprise 가격 |
| **Fastly** | edge compute (VCL) | 학습 곡선 |
| **Akamai** | enterprise / 매우 큰 규모 | 비싸 |
| **CloudFlare R2 + Pages** | object storage + static | newer |
| **Vercel / Netlify** | static / Next.js 친화 | 일반 backend 약함 |
| **GCP Cloud CDN** | GCP LB 통합 | 단독 사용 어려움 |

---

## 3. 흐름

```
[user] → DNS → [CDN POP (edge)] → (cache miss) → [origin]
                    │
                    └─ (cache hit) → 즉시 응답
```

cache key = URL + (선택) query / cookie / header.

---

## 4. cache 정책

```
1. CDN 의 TTL (Cache-Control max-age 존중 or override)
2. cache key 정규화 (query 정렬 / lowercase)
3. immutable static (이미지 / JS hash 파일) — 1년
4. dynamic API — short (1min) 또는 no-cache
```

```http
# origin response header
Cache-Control: public, max-age=31536000, immutable
Cache-Control: private, max-age=60
Cache-Control: no-cache, must-revalidate
```

→ **hash 파일명 (`app.abc123.js`) + immutable** = 최고의 cache.

---

## 5. CloudFront 예

```yaml
# distribution
origins:
  - id: alb
    domain: alb.example.com
    custom-origin: {protocol-policy: https-only}
  - id: s3
    domain: bucket.s3.amazonaws.com
    s3-origin-config: {origin-access-identity: ...}

cache-behaviors:
  - path-pattern: /static/*
    target-origin: s3
    viewer-protocol-policy: redirect-to-https
    cache-policy: CachingOptimized
    compress: true

  - path-pattern: /api/*
    target-origin: alb
    cache-policy: CachingDisabled
    origin-request-policy: AllViewer

default-cache-behavior:
  target-origin: alb
  cache-policy: CachingOptimized
  viewer-protocol-policy: redirect-to-https
```

---

## 6. invalidation (cache 비우기)

```bash
# CloudFront
aws cloudfront create-invalidation \
    --distribution-id E1234 \
    --paths "/index.html" "/static/*"
```

```bash
# Cloudflare
curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE/purge_cache" \
    -H "Authorization: Bearer $TOKEN" \
    -d '{"files": ["https://example.com/index.html"]}'
```

→ **invalidation 은 비싸고 느림**. 가능하면 hash 파일명 + immutable.

---

## 7. signed URL / cookie (private content)

```python
# CloudFront signed URL
url = generate_signed_url(
    resource="https://cdn.example.com/private/video.mp4",
    expires_at=time + 1*hour,
    private_key=key
)
```

→ 인증된 사용자만 접근. 동영상 / paid content.

---

## 8. edge compute (★)

```javascript
// CloudFront Functions (간단)
function handler(event) {
    var request = event.request;
    var uri = request.uri;
    if (uri.endsWith('/')) request.uri = uri + 'index.html';
    return request;
}

// Lambda@Edge (강력)
exports.handler = async (event) => {
    const request = event.Records[0].cf.request;
    // A/B test, auth, geo routing, image resize
    return request;
};

// Cloudflare Workers (가장 강력)
addEventListener('fetch', event => {
    event.respondWith(handle(event.request));
});
```

→ origin 호출 없이 edge 에서 처리. latency 매우 낮음.

---

## 9. WAF / DDoS

```
CDN 단에서 차단:
- WAF rule (SQL injection / XSS / OWASP CRS)
- DDoS (Cloudflare 무료 무제한 / AWS Shield Standard 무료)
- rate-limit
- bot management
- geo-block
```

→ origin 까지 안 옴.

---

## 10. observability

```
- access log → S3 / Cloudflare R2
- real-user monitoring (RUM)
- cache hit rate (★ 핵심 KPI)
- origin shield (single point cache before origin)
```

→ cache hit rate < 80% 면 정책 점검.

---

## 11. CDN 없이 / 자체 cache

```
nginx proxy_cache    — 단일 region, 작은 규모 OK
Varnish              — Fastly 의 base
Squid                — legacy
```

→ 글로벌 / 대규모 = CDN 필수.

---

## 12. 함정

1. **dynamic API cache** — 사용자별 다른 응답이 같은 cache → 데이터 누출.
2. **cache key 에 user 정보 누락** — 위와 동일.
3. **TTL 너무 김** — 변경 안 반영. invalidation 비쌈.
4. **TTL 너무 짧음** — origin 부담.
5. **vary header 무시** — Accept-Language 다른데 같은 응답.
6. **gzip / brotli 미지원** origin — bandwidth 낭비.
7. **HTTPS origin 아님** — CDN ↔ origin 평문 (보안 위험).

---

## 13. 관련

- [[networking-ops|↑ networking-ops]]
- [[waf]]
- [[../nginx/caching|↗ nginx cache]]
