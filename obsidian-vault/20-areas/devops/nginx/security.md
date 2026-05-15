---
title: "nginx 보안 — header / WAF / hardening"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:24:00+09:00
tags: [devops, nginx, security]
---

# nginx 보안 — header / WAF / hardening

**[[nginx|↑ nginx]]**

---

## 1. 보안 header (★)

```nginx
# HSTS — HTTPS 강제 (1년 잠금. 신중히)
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

# XSS / content type
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "0" always;        # 현재 권장 = 0 (legacy)

# CSP (Content Security Policy)
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self'" always;

# Referrer
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# Permissions Policy
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

# COOP / COEP (cross-origin isolation, SharedArrayBuffer 등)
add_header Cross-Origin-Opener-Policy "same-origin" always;
add_header Cross-Origin-Embedder-Policy "require-corp" always;
```

→ **`always`** 가 중요: 4xx/5xx 응답에도 적용.

---

## 2. 정보 누출 방지

```nginx
server_tokens off;                    # Server: nginx (version 숨김)
more_clear_headers Server;            # nginx-extras

# backend header 숨김
proxy_hide_header X-Powered-By;
proxy_hide_header Server;
proxy_hide_header X-AspNet-Version;
```

→ Nginx version exposure → 공격자에게 known vuln 정보.

---

## 3. 위험한 method 제한

```nginx
location / {
    if ($request_method !~ ^(GET|POST|PUT|DELETE|PATCH|OPTIONS|HEAD)$) {
        return 405;
    }
    proxy_pass http://backend;
}
```

→ TRACE / TRACK / CONNECT 차단.

---

## 4. user-agent / referer 필터

```nginx
# 의심스러운 UA 차단
if ($http_user_agent ~* "(nikto|sqlmap|nessus|nmap)") {
    return 403;
}

# 빈 UA 차단
if ($http_user_agent = "") {
    return 403;
}

# hotlinking 방지
location ~* \.(jpg|png|gif)$ {
    valid_referers none blocked example.com *.example.com;
    if ($invalid_referer) {
        return 403;
    }
}
```

→ **`if` 는 신중히** ("if is evil" docs). 가능하면 map.

---

## 5. body / URL 제한

```nginx
client_max_body_size 10m;             # upload 크기 제한
large_client_header_buffers 4 16k;   # header 크기

# slowloris 방지
client_body_timeout 10s;
client_header_timeout 10s;
keepalive_timeout 65s;
send_timeout 10s;
```

---

## 6. IP allow / deny

```nginx
location /admin/ {
    allow 10.0.0.0/8;
    allow 192.168.0.0/16;
    deny all;

    proxy_pass http://admin-backend;
}
```

→ `/admin/` 은 내부망 만.

---

## 7. ModSecurity (WAF)

```bash
sudo apt install libnginx-mod-http-modsecurity

# /etc/nginx/modsec/main.conf
SecRuleEngine On
Include /etc/nginx/modsec/coreruleset/crs-setup.conf
Include /etc/nginx/modsec/coreruleset/rules/*.conf
```

```nginx
# nginx.conf
http {
    modsecurity on;
    modsecurity_rules_file /etc/nginx/modsec/main.conf;
}
```

→ OWASP Core Rule Set (CRS) 으로 SQL injection / XSS / RCE 등 차단.

---

## 8. naxsi (다른 WAF)

```nginx
location / {
    SecRulesEnabled;
    DeniedUrl "/RequestDenied";
    CheckRule "$SQL >= 8" BLOCK;
    CheckRule "$RFI >= 8" BLOCK;
    CheckRule "$TRAVERSAL >= 4" BLOCK;
    CheckRule "$EVADE >= 4" BLOCK;
    CheckRule "$XSS >= 8" BLOCK;
    proxy_pass http://backend;
}
```

→ ModSecurity 보다 가벼움.

---

## 9. fail2ban 통합

```ini
# /etc/fail2ban/jail.local
[nginx-limit-req]
enabled = true
filter = nginx-limit-req
action = iptables-multiport[name=ReqLimit, port="http,https"]
logpath = /var/log/nginx/error.log
findtime = 600
bantime = 7200
maxretry = 10
```

→ nginx rate limit 로그 → fail2ban → iptables 차단.

---

## 10. 권장 hardening 체크리스트

```
☐ TLS 1.2+ only, modern cipher
☐ HSTS preload (신중히)
☐ CSP / X-Frame-Options / X-Content-Type-Options
☐ server_tokens off
☐ proxy_hide_header (X-Powered-By 등)
☐ method 제한 (GET/POST/...)
☐ client_max_body_size 합리적
☐ rate-limit (limit_req)
☐ /admin allow 내부망
☐ access_log 보존 (audit)
☐ /etc/nginx 권한 700 / files 644
☐ nginx user (non-root)
☐ ModSecurity / WAF
☐ fail2ban
☐ 정기 update (`apt upgrade nginx`)
```

---

## 11. CSP 디자인 팁

```nginx
# default 엄격 → 점진 완화
Content-Security-Policy: default-src 'none';
                         script-src 'self';
                         style-src 'self';
                         img-src 'self' data:;
                         connect-src 'self';
                         font-src 'self';
                         frame-ancestors 'none';
                         base-uri 'self';
                         form-action 'self';
                         report-uri /csp-report

# 운영 시작: Content-Security-Policy-Report-Only — 차단 X, 위반만 보고
# → 검증 후 enforce.
```

---

## 12. 함정

1. **HSTS preload — 잘못 등록** — 1년 잠김 (해제 어려움).
2. **CSP 너무 엄격 → site 깨짐** — Report-Only 로 먼저.
3. **`if` 남용** — 의도와 다른 동작.
4. **server_tokens on** (default) — version 노출.
5. **WAF 켜고 검증 안 함** — false positive → 정상 traffic 차단.
6. **`add_header always` 빠짐** — 4xx 응답에서 header 빠짐.

---

## 13. 관련

- [[nginx|↑ nginx]]
- [[tls-https]]
- [[rate-limiting]]
- [[../security-ops/security-ops|↗ security-ops]]
