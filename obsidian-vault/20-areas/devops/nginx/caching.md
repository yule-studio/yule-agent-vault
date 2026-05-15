---
title: "nginx caching — proxy_cache"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:20:00+09:00
tags: [devops, nginx, cache]
---

# nginx caching — proxy_cache

**[[nginx|↑ nginx]]**

---

## 1. 왜

- backend 부담 ↓ (static / 잘 변하지 않는 API).
- latency ↓ (디스크 / 메모리 hit).
- CDN 대안 (작은 규모).

→ 대용량 글로벌 = CDN (CloudFront / Cloudflare). 일반 = nginx cache 충분.

---

## 2. 기본 설정

```nginx
# http {} context
proxy_cache_path /var/cache/nginx
    levels=1:2
    keys_zone=my_cache:10m         # 10MB metadata (≈ 80,000 key)
    max_size=10g                    # cache 디스크 max
    inactive=60m                    # 60min 미사용 시 제거
    use_temp_path=off;
```

```nginx
# location 에서 사용
location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 302 10m;
    proxy_cache_valid 404 1m;
    proxy_cache_key "$scheme$request_method$host$request_uri";
    proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
    proxy_cache_lock on;
    proxy_pass http://backend;

    add_header X-Cache-Status $upstream_cache_status;
}
```

---

## 3. 응답 header 확인

```
X-Cache-Status: HIT / MISS / BYPASS / EXPIRED / STALE / UPDATING / REVALIDATED
```

`curl -I https://example.com/page` 으로 확인.

---

## 4. cache key 디자인

```nginx
# Default
proxy_cache_key $scheme$proxy_host$request_uri;

# 자주 사용
proxy_cache_key "$scheme$request_method$host$request_uri";

# 쿠키 / 사용자별
proxy_cache_key "$scheme$host$request_uri|$cookie_user_tier";

# header 별
proxy_cache_key "$scheme$host$request_uri|$http_accept_language";
```

→ key 너무 자세하면 cache hit 율 낮음. 너무 단순하면 잘못된 응답.

---

## 5. cache 우회 조건

```nginx
# 특정 쿠키 / header 있으면 cache 안 함
proxy_no_cache $cookie_session $http_authorization;
proxy_cache_bypass $cookie_session $http_authorization;

# query param 으로 우회
location / {
    set $skip_cache 0;
    if ($arg_nocache) { set $skip_cache 1; }
    proxy_cache_bypass $skip_cache;
    proxy_no_cache $skip_cache;
    proxy_pass http://backend;
}
```

---

## 6. cache TTL

```nginx
# response status 별
proxy_cache_valid 200 302 1h;
proxy_cache_valid 404 1m;
proxy_cache_valid any 1m;        # 모든 status 1분

# 또는 backend 의 Cache-Control 존중
proxy_cache_valid 200 1h;
proxy_ignore_headers Cache-Control;    # backend header 무시
# 또는 사용
# (backend 의 Expires/Cache-Control 따름이 default)
```

---

## 7. cache lock (thundering herd 방지)

```nginx
proxy_cache_lock on;
proxy_cache_lock_timeout 5s;
```

→ 같은 key 가 cache miss 일 때, 한 request 만 backend 호출. 다른 request 는 대기.

---

## 8. stale-while-revalidate

```nginx
proxy_cache_use_stale
    error
    timeout
    updating
    http_500 http_502 http_503 http_504;

proxy_cache_background_update on;
proxy_cache_revalidate on;
```

→ backend 가 down 이어도 stale 응답. 살아 있으면 background 갱신.

---

## 9. cache purge

```nginx
# OSS: 직접 디렉터리 삭제
sudo find /var/cache/nginx -type f -delete
sudo systemctl reload nginx

# nginx Plus: proxy_cache_purge directive
location ~ /purge(/.*) {
    allow 127.0.0.1;
    deny all;
    proxy_cache_purge my_cache $scheme$host$1;
}
```

---

## 10. micro-caching (1초 cache)

```nginx
# trade-off: stale 1s 허용으로 backend 폭주 방지
location / {
    proxy_cache my_cache;
    proxy_cache_valid 200 1s;
    proxy_cache_lock on;
    proxy_pass http://backend;
}
```

→ traffic spike 시 효과적. 1초 stale 은 보통 OK.

---

## 11. static file 직접 service

```nginx
# proxy_cache 보다 native serving 이 빠름
location /static/ {
    alias /var/www/static/;
    expires 1y;
    add_header Cache-Control "public, immutable";
}

location ~* \.(jpg|jpeg|png|gif|webp|svg|ico|woff2)$ {
    expires 30d;
    add_header Cache-Control "public";
    access_log off;
}
```

→ CSS / JS 는 hash 파일명 (`app.abc123.js`) → `immutable` OK.

---

## 12. 함정

1. **cache key 에 user 정보 누락** — 다른 사용자 데이터 누출 (★ 위험).
2. **POST cache 됨** — `proxy_cache_methods GET HEAD;` (default OK).
3. **set-cookie 응답 cache** — 같은 cookie 가 모든 사용자에게.
4. **cache 무한 — backend 변경 안 됨** — `proxy_cache_valid` + purge 전략.
5. **disk 부족** — `max_size` + 모니터링.
6. **cache lock + 느린 backend** — 다른 request 가 timeout.

---

## 13. 관련

- [[nginx|↑ nginx]]
- [[reverse-proxy]]
- [[performance-tuning]]
