---
title: "nginx config 구조 + location 매칭"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:10:00+09:00
tags: [devops, nginx, config]
---

# nginx config 구조 + location 매칭

**[[nginx|↑ nginx]]**

---

## 1. 디렉터리 구조 (Debian/Ubuntu)

```
/etc/nginx/
├── nginx.conf                 # 메인
├── conf.d/                    # http {} 안에 include 됨
│   └── default.conf
├── sites-available/           # 가상 호스트 (만 있는 곳)
│   ├── example.com.conf
│   └── api.example.com.conf
├── sites-enabled/             # enabled (symlink)
│   └── example.com.conf -> ../sites-available/example.com.conf
├── snippets/                  # 재사용
│   ├── ssl-params.conf
│   └── fastcgi-params
└── modules-enabled/
```

→ enable: `ln -s ../sites-available/X.conf ./sites-enabled/`

→ RHEL 계열은 보통 `/etc/nginx/conf.d/` 만 사용.

---

## 2. 가상 호스트 (server block)

```nginx
# /etc/nginx/sites-available/example.com.conf
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # HTTPS redirect
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    include snippets/ssl-params.conf;

    root /var/www/example.com;
    index index.html;

    access_log /var/log/nginx/example.com.access.log;
    error_log /var/log/nginx/example.com.error.log;

    location / {
        try_files $uri $uri/ =404;
    }

    location /api/ {
        proxy_pass http://backend;
        include snippets/proxy-headers.conf;
    }
}
```

---

## 3. server_name 매칭 우선순위

```
1. 정확히 일치 (example.com)
2. 와일드카드 앞 (*.example.com)
3. 와일드카드 뒤 (example.*)
4. 정규식 (~^(www\.)?example\.com$)
5. default_server
```

```nginx
listen 80 default_server;       # 어디에도 매치 안 되면 여기로
server_name _;                  # catch-all
```

---

## 4. location 매칭 (★)

```nginx
location [modifier] /pattern { ... }
```

| modifier | 의미 | 우선순위 |
| --- | --- | --- |
| `=` | 정확히 일치 | 1 (최우선) |
| `^~` | prefix (regex 무시) | 2 |
| `~` | 정규식 (case-sensitive) | 3 |
| `~*` | 정규식 (case-insensitive) | 3 |
| (없음) | prefix (가장 긴 매칭) | 4 |

```nginx
# 예
location = /favicon.ico { ... }    # /favicon.ico 만
location ^~ /static/ { ... }       # /static/* (regex 무시)
location ~* \.(jpg|png)$ { ... }   # 확장자 regex
location /api/ { ... }             # prefix
location / { ... }                 # catch-all
```

---

## 5. proxy headers (snippet)

```nginx
# /etc/nginx/snippets/proxy-headers.conf
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Port $server_port;

proxy_http_version 1.1;
proxy_set_header Connection "";          # keepalive

proxy_connect_timeout 5s;
proxy_send_timeout 60s;
proxy_read_timeout 60s;
```

---

## 6. SSL snippet

```nginx
# /etc/nginx/snippets/ssl-params.conf
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:...;
ssl_prefer_server_ciphers off;
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;

ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 8.8.8.8 valid=300s;
resolver_timeout 5s;

# HSTS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

---

## 7. include 패턴

```nginx
http {
    # mime type
    include mime.types;
    default_type application/octet-stream;

    # 공통 log format
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    'rt=$request_time uct=$upstream_connect_time '
                    'urt=$upstream_response_time';
    access_log /var/log/nginx/access.log main;

    # 가상호스트 자동 include
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

---

## 8. try_files (SPA / static)

```nginx
# SPA (React / Vue)
location / {
    try_files $uri $uri/ /index.html;     # 못 찾으면 index.html (SPA routing)
}

# 정적 파일 우선, 없으면 backend
location / {
    try_files $uri @backend;
}
location @backend {
    proxy_pass http://app;
}
```

---

## 9. error / redirect

```nginx
error_page 404 /404.html;
error_page 500 502 503 504 /50x.html;

location = /50x.html {
    root /var/www/errors;
    internal;
}

# redirect
location /old-path {
    return 301 https://example.com/new-path;
}

# rewrite (regex)
rewrite ^/old/(.*)$ /new/$1 permanent;
```

---

## 10. 함정

1. **sites-enabled 의 default 남음** — 다른 server block 매칭 X.
2. **server_name 따옴표** — `server_name "example.com"` ❌, `server_name example.com;`.
3. **listen `[::]:80` 빠짐** — IPv6 미동작.
4. **default_server 두 번** — error.
5. **location regex 우선순위 오해** — exact > prefix `^~` > regex.
6. **trailing slash** — `proxy_pass http://backend/` 와 `http://backend` 다름.

---

## 11. 관련

- [[nginx|↑ nginx]]
- [[concepts]]
- [[reverse-proxy]]
