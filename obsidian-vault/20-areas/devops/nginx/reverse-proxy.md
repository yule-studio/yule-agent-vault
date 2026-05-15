---
title: "Reverse proxy — proxy_pass + header"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:13:00+09:00
tags: [devops, nginx, proxy]
---

# Reverse proxy — proxy_pass + header

**[[nginx|↑ nginx]]**

---

## 1. 가장 단순한 reverse proxy

```nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://localhost:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

→ `api.example.com` 의 모든 요청 → `localhost:8080` 으로.

---

## 2. proxy_pass 의 trailing slash (★)

```nginx
# A. proxy_pass http://backend/    (slash 있음)
location /api/ {
    proxy_pass http://backend/;
}
# 요청 /api/users → backend 로 /users      ← URI 의 /api/ 가 잘림

# B. proxy_pass http://backend     (slash 없음)
location /api/ {
    proxy_pass http://backend;
}
# 요청 /api/users → backend 로 /api/users  ← 전체 URI 그대로
```

→ **slash 차이로 URI rewrite 됨**. 가장 헷갈리는 부분.

---

## 3. 표준 proxy header

```nginx
proxy_set_header Host $host;                         # 원래 host
proxy_set_header X-Real-IP $remote_addr;             # client IP
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;          # http / https
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port $server_port;

# WebSocket / SSE
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

→ Spring 의 `ForwardedHeaderFilter` / `server.forward-headers-strategy: framework` 와 짝.

---

## 4. timeout

```nginx
proxy_connect_timeout 5s;    # backend 와 connect 시도
proxy_send_timeout 60s;      # backend 로 body 전송 시간
proxy_read_timeout 60s;      # backend 응답 대기

# 긴 SSE / WebSocket
location /ws {
    proxy_pass http://backend;
    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

---

## 5. keepalive (backend 연결 재사용)

```nginx
upstream backend {
    server app1:8080;
    server app2:8080;
    keepalive 32;             # ★ 유지할 connection 수
}

server {
    location / {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";    # ★ keepalive 동작 필수
    }
}
```

→ keepalive 없으면 매 요청마다 새 TCP connection → 큰 overhead.

---

## 6. proxy_pass 대상

```nginx
# IP
proxy_pass http://10.0.0.5:8080;

# DNS (이름)
proxy_pass http://backend.internal:8080;
# → resolve 는 startup 시 한 번만 (★ 함정)

# 동적 resolve (변동 IP 환경 — k8s 서비스)
resolver 10.96.0.10 valid=10s;
set $upstream "backend.svc.cluster.local";
proxy_pass http://$upstream:8080;
# → 변수 사용 시 매 요청 resolve

# upstream 이름 (load balance)
proxy_pass http://my-backend;
```

---

## 7. URI rewrite + proxy

```nginx
# /old/* → backend 의 /new/*
location /old/ {
    rewrite ^/old/(.*)$ /new/$1 break;
    proxy_pass http://backend;
}

# /v1/* → backend (path 유지)
location /v1/ {
    proxy_pass http://backend;
}

# /api/ → backend root
location /api/ {
    proxy_pass http://backend/;
}
```

---

## 8. response 변경

```nginx
# header 추가/제거
proxy_hide_header X-Powered-By;
proxy_hide_header Server;
add_header X-Custom "value" always;

# body rewrite (sub_filter)
location / {
    proxy_pass http://backend;
    sub_filter '<title>old</title>' '<title>new</title>';
    sub_filter_once on;
    proxy_set_header Accept-Encoding "";  # gzip 비활성 (sub_filter 동작 위해)
}
```

---

## 9. error 처리

```nginx
proxy_intercept_errors on;            # backend 의 5xx 가로채기

error_page 502 503 504 /maintenance.html;
location = /maintenance.html {
    root /var/www/errors;
    internal;
}

# upstream 모두 fail 시 fallback
location / {
    proxy_pass http://backend;
    proxy_next_upstream error timeout http_502 http_503;
    proxy_next_upstream_tries 3;
    proxy_next_upstream_timeout 10s;
}
```

---

## 10. health check (open source nginx 한계)

```nginx
# OSS nginx 는 passive health check 만 (요청 실패 시 mark down)
upstream backend {
    server app1:8080 max_fails=3 fail_timeout=30s;
    server app2:8080 max_fails=3 fail_timeout=30s;
    server app3:8080 backup;           # backup (active 모두 fail 시)
}
```

→ active health check 는 nginx Plus 또는 OpenResty + lua.

---

## 11. proxy_buffering

```nginx
# default on — backend 응답 전체 받아서 client 로
proxy_buffering on;
proxy_buffer_size 4k;
proxy_buffers 8 8k;

# off — backend → client 즉시 stream (SSE / 다운로드)
location /sse {
    proxy_pass http://backend;
    proxy_buffering off;
    proxy_cache off;
}
```

---

## 12. 함정

1. **trailing slash 차이** — `proxy_pass http://b/;` vs `http://b;` 동작 다름.
2. **Host header 안 넘김** — backend 가 잘못된 host 인식.
3. **X-Forwarded-For 신뢰** — 외부 직접 접속 시 spoofable. **real_ip_header + set_real_ip_from** 사용.
4. **DNS 한 번만 resolve** — 변수 (`$upstream`) 또는 resolver 필요.
5. **keepalive Connection 미설정** — connection 재사용 X.
6. **proxy_buffering on + SSE** — client 가 chunk 늦게 받음.
7. **timeout default (60s) 너무 짧음** — long-polling 끊김.

---

## 13. 관련

- [[nginx|↑ nginx]]
- [[load-balancing]]
- [[config-structure]]
- [[caching]]
