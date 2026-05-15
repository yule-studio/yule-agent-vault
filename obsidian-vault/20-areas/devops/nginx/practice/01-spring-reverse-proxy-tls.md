---
title: "실습 01 — Spring + nginx reverse proxy + TLS"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:32:00+09:00
tags: [devops, nginx, practice, tls, spring]
---

# 실습 01 — Spring + nginx reverse proxy + TLS

**[[practice|↑ practice]]**

---

## 목표

- Ubuntu VM 에 nginx 설치
- Spring Boot 앱을 localhost:8080 에 실행
- nginx 가 443 / 80 받아 reverse proxy
- Let's Encrypt 로 자동 HTTPS

---

## 1. 사전

- Ubuntu 22.04 server
- 공인 IP + 도메인 (DuckDNS `myapp.duckdns.org` 추천)
- DNS A record → 서버 IP

---

## 2. nginx 설치

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx

curl http://localhost/        # default page 확인
```

---

## 3. Spring 실행

```bash
# 샘플 (Day 01 의 demo-web)
java -jar demo-web.jar         # localhost:8080
curl http://localhost:8080/hello
```

systemd unit 으로 영구 실행:

```ini
# /etc/systemd/system/demo-web.service
[Unit]
Description=Demo Web
After=network.target

[Service]
User=demo
ExecStart=/usr/bin/java -jar /opt/demo/demo-web.jar
Restart=on-failure
Environment="SERVER_PORT=8080"

[Install]
WantedBy=multi-user.target
```

```bash
sudo useradd --system --no-create-home demo
sudo cp demo-web.jar /opt/demo/
sudo chown -R demo /opt/demo
sudo systemctl daemon-reload
sudo systemctl enable --now demo-web
```

---

## 4. nginx HTTP 설정

```nginx
# /etc/nginx/sites-available/demo
server {
    listen 80;
    server_name myapp.duckdns.org;

    location / {
        proxy_pass http://localhost:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/demo /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx

curl http://myapp.duckdns.org/hello       # → "world"
```

---

## 5. HTTPS — certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d myapp.duckdns.org

# certbot 자동:
# - 인증서 발급
# - nginx config 수정 (443 + ssl_certificate 추가)
# - HTTP → HTTPS redirect 추가

curl https://myapp.duckdns.org/hello
```

자동 갱신 확인:
```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

---

## 6. ssl-params 적용 (modern)

```bash
sudo cp ~/snippets/ssl-params.conf /etc/nginx/snippets/
```

`/etc/nginx/sites-available/demo` 의 443 block 안:
```nginx
include snippets/ssl-params.conf;
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 7. SSL Labs 테스트

브라우저에서:
```
https://www.ssllabs.com/ssltest/analyze.html?d=myapp.duckdns.org
```

→ A+ 목표.

---

## 8. 보안 헤더 추가

```nginx
# 443 server block 안
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
server_tokens off;
```

---

## 9. 검증 체크리스트

- [ ] `http://myapp.duckdns.org` → 자동 301 → https
- [ ] `https://myapp.duckdns.org/hello` → "world"
- [ ] `curl -I https://myapp.duckdns.org` → HSTS / X-Frame-Options 보임
- [ ] SSL Labs A+ 또는 A
- [ ] `sudo certbot renew --dry-run` 성공
- [ ] Server header 에 nginx version 없음

---

## 10. 관련

- [[practice|↑ practice]]
- [[../tls-https]]
- [[../reverse-proxy]]
- [[../security]]
