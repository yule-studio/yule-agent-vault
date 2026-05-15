---
title: "TLS / HTTPS — certbot + Let's Encrypt"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:18:00+09:00
tags: [devops, nginx, tls, https]
---

# TLS / HTTPS — certbot + Let's Encrypt

**[[nginx|↑ nginx]]**

---

## 1. 인증서 발급 (Let's Encrypt + certbot)

```bash
# 설치
sudo apt install certbot python3-certbot-nginx

# 발급 + nginx 자동 설정
sudo certbot --nginx -d example.com -d www.example.com

# 또는 standalone (nginx stop 후)
sudo certbot certonly --standalone -d example.com

# 또는 DNS challenge (wildcard 가능)
sudo certbot certonly --dns-cloudflare \
    --dns-cloudflare-credentials ~/.secrets/cloudflare.ini \
    -d "*.example.com" -d example.com
```

발급 위치:
```
/etc/letsencrypt/live/example.com/
├── fullchain.pem        # 체인 (서버에서 사용)
├── privkey.pem          # private key
├── cert.pem             # 서버 cert
└── chain.pem            # intermediate
```

---

## 2. 자동 갱신

```bash
# 90일 인증서 → 30일 전 자동 갱신
sudo systemctl status certbot.timer       # 또는 cron

# 수동 dry-run
sudo certbot renew --dry-run

# 갱신 후 nginx reload
# /etc/letsencrypt/renewal-hooks/post/reload-nginx.sh
#!/bin/bash
systemctl reload nginx
```

---

## 3. nginx HTTPS 설정

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    # HTTP → HTTPS redirect
    location / {
        return 301 https://$host$request_uri;
    }

    # ACME challenge (certbot 갱신)
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name example.com www.example.com;

    # 인증서
    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    include snippets/ssl-params.conf;

    location / {
        proxy_pass http://backend;
    }
}
```

---

## 4. ssl-params.conf (modern)

```nginx
# /etc/nginx/snippets/ssl-params.conf

# 프로토콜
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;          # TLS 1.3 권장 (client 우선)

# Cipher (Mozilla intermediate)
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;

# Session
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;

# OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;
ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;
resolver 1.1.1.1 8.8.8.8 valid=300s;

# HSTS (★ 신중히 — 1년 잠금)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# 기타 보안 header
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Diffie-Hellman (optional, 4096-bit)
# ssl_dhparam /etc/nginx/dhparam.pem;
# openssl dhparam -out /etc/nginx/dhparam.pem 4096
```

---

## 5. Mozilla SSL Config Generator

```
https://ssl-config.mozilla.org/
```

→ Server / OpenSSL 버전 입력 → nginx config 자동 생성. modern / intermediate / old.

---

## 6. 검증 / 테스트

```bash
# 서버에서
sudo nginx -t
openssl s_client -connect example.com:443 -servername example.com
openssl s_client -connect example.com:443 -showcerts

# 외부 도구
curl -vk https://example.com
testssl.sh example.com                # 종합 분석
nmap --script ssl-enum-ciphers -p 443 example.com

# SSL Labs (★ 표준)
https://www.ssllabs.com/ssltest/analyze.html?d=example.com
# → A+ 목표
```

---

## 7. mutual TLS (mTLS)

```nginx
server {
    listen 443 ssl http2;

    # 서버 cert
    ssl_certificate     /etc/nginx/server.crt;
    ssl_certificate_key /etc/nginx/server.key;

    # client cert 검증
    ssl_client_certificate /etc/nginx/ca.crt;
    ssl_verify_client on;                    # 또는 optional
    ssl_verify_depth 2;

    location / {
        proxy_set_header X-SSL-Client-Subject $ssl_client_s_dn;
        proxy_set_header X-SSL-Client-Verify  $ssl_client_verify;
        proxy_pass http://backend;
    }
}
```

→ B2B API / 내부 service 간.

---

## 8. SNI (Server Name Indication)

```
한 IP : 여러 도메인 가능. nginx 가 ClientHello 의 server_name 으로 server block 결정.
default — TLS 1.0+ 부터 표준.
```

```nginx
server {
    listen 443 ssl;
    server_name a.example.com;
    ssl_certificate /etc/letsencrypt/live/a.example.com/fullchain.pem;
    # ...
}

server {
    listen 443 ssl;
    server_name b.example.com;
    ssl_certificate /etc/letsencrypt/live/b.example.com/fullchain.pem;
    # ...
}
```

---

## 9. ACME DNS-01 (wildcard)

```bash
# *.example.com 인증서 → DNS-01 만 가능 (HTTP-01 불가)
sudo certbot certonly --manual --preferred-challenges dns -d "*.example.com"
# → 안내된 TXT 레코드 추가 → 검증

# 자동화 (Cloudflare)
sudo certbot certonly \
    --dns-cloudflare \
    --dns-cloudflare-credentials ~/cloudflare.ini \
    -d "*.example.com" -d example.com
```

---

## 10. 함정

1. **HSTS 너무 일찍 켜기** — 잘못된 cert 시 1년 잠김.
2. **TLS 1.0/1.1 켜둠** — 취약.
3. **자체서명 cert** — 브라우저 경고.
4. **갱신 hook 없음** — cert 갱신 후 nginx 가 옛 cert 사용.
5. **ACME challenge path block** — `/.well-known/acme-challenge/` 막힘 → 갱신 실패.
6. **`ssl_certificate` 만, intermediate 없음** — 일부 client 가 cert 못 검증.
7. **rate limit (Let's Encrypt)** — 같은 도메인 주당 5회 시도 한도.

---

## 11. 관련

- [[nginx|↑ nginx]]
- [[security]]
- [[../security-ops/security-ops|↗ security-ops]]
