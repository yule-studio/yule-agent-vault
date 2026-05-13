---
title: "curl / httpie — HTTP 클라이언트"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:10:00+09:00
tags:
  - network
  - tools
  - curl
  - http
---

# curl / httpie — HTTP 클라이언트

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | curl 옵션 / httpie / 디버깅 |

**[[tools|↑ 도구 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. curl — 가장 강력 / 보편

### 기본
```bash
curl https://example.com
curl -I https://example.com           # HEAD (헤더만)
curl -v https://example.com           # verbose
curl -L https://example.com           # follow redirects
curl -o file.html https://...         # 저장
curl -O https://example.com/file.zip  # 같은 이름 저장
```

### HTTP method
```bash
curl -X POST https://api/users -d 'name=alice'
curl -X PUT https://api/users/1 -d 'name=alice'
curl -X DELETE https://api/users/1
curl -X PATCH https://api/users/1 -d 'name=bob'
```

### Header
```bash
curl -H "Content-Type: application/json" \
     -H "Authorization: Bearer xyz" \
     https://api
```

### JSON body
```bash
curl -X POST -H "Content-Type: application/json" \
     -d '{"name": "alice"}' \
     https://api/users

# 파일에서
curl -X POST -H "Content-Type: application/json" \
     -d @body.json \
     https://api/users
```

### Form data
```bash
# x-www-form-urlencoded
curl -d "name=alice&email=alice@a.com" https://api/users

# multipart
curl -F "name=alice" -F "avatar=@photo.jpg" https://api/users
```

### Cookie
```bash
curl -c cookies.txt -b cookies.txt https://example.com    # save + send
curl -b "session=abc" https://example.com
```

### Auth
```bash
curl -u user:pass https://api          # Basic
curl --digest -u user:pass https://api  # Digest
curl -H "Authorization: Bearer xyz" https://api
```

### Proxy
```bash
curl -x http://proxy:8080 https://example.com
curl --socks5 localhost:1080 https://example.com
```

### TLS
```bash
# Cert 검증 무시
curl -k https://self-signed.example.com

# Client cert
curl --cert client.pem --key client.key https://api

# TLS version
curl --tlsv1.2 https://example.com
curl --tls-max 1.2 https://example.com
```

### HTTP version
```bash
curl --http1.0 https://example.com
curl --http1.1 https://example.com
curl --http2 https://example.com
curl --http3 https://example.com         # curl 7.66+
```

### --resolve — DNS 우회
```bash
curl --resolve example.com:443:1.2.3.4 https://example.com
```

→ DNS 안 변경하고 IP 테스트.

### 자세한 정보 — verbose / trace
```bash
curl -v https://example.com
curl --trace-ascii dump.txt https://example.com
curl -w "%{http_code} %{time_total}\n" -o /dev/null https://example.com
```

### Timing breakdown
```bash
curl -w "@-" -o /dev/null -s https://example.com <<'EOF'
DNS:        %{time_namelookup}
Connect:    %{time_connect}
TLS:        %{time_appconnect}
TTFB:       %{time_starttransfer}
Total:      %{time_total}
EOF
```

### 재시도 / 속도
```bash
curl --retry 3 --retry-delay 1 https://api
curl --max-time 10 https://api
curl --limit-rate 100K https://example.com/big-file
```

### Range
```bash
curl -r 0-499 https://example.com/file
curl --range 1000- https://example.com/file
```

---

## 2. httpie — 더 친화

```bash
# 기본 (GET)
http https://example.com

# POST + JSON 자동
http POST https://api/users name=alice email=alice@a.com

# Header
http https://api Authorization:'Bearer xyz' Content-Type:application/json

# Query
http https://api q==search page==1

# Form
http --form POST https://api/users name=alice avatar@photo.jpg

# Stream / output
http --download https://example.com/big.zip

# Pretty
http https://example.com    # JSON pretty 자동
```

### 장점
- 직관적
- JSON 자동
- 컬러
- Session (`--session=mysession`)

### 단점
- 옛 / 잘 알려진 — curl

---

## 3. Postman / Insomnia / Bruno

### GUI 도구
- **Postman** — 가장 보편 / cloud sync
- **Insomnia** — open source / 가볍
- **Bruno** — local-first / git-friendly
- **Hoppscotch** — web-based

### 사용
- Collection 관리
- 환경 변수
- Pre/post script
- Mock server
- API 문서

---

## 4. 흔한 시나리오

### 4.1 API 디버깅
```bash
curl -v -H "Authorization: Bearer $TOKEN" https://api/me
```

### 4.2 Redirect 추적
```bash
curl -L -v https://goo.gl/abc 2>&1 | grep '< Location:'
```

### 4.3 다른 IP 로 같은 호스트
```bash
# Production vs staging — 같은 도메인, 다른 IP
curl --resolve example.com:443:1.2.3.4 https://example.com    # prod
curl --resolve example.com:443:5.6.7.8 https://example.com    # staging
```

### 4.4 GeoIP 차이
```bash
curl --interface 1.2.3.4 https://api/geo
# 또는 다른 region 의 server 에서
```

### 4.5 HTTPS handshake
```bash
curl -v --tlsv1.3 https://example.com 2>&1 | grep -i 'tls\|cipher'
```

### 4.6 File upload
```bash
curl -F "file=@photo.jpg" -F "title=My Photo" https://api/upload
```

### 4.7 Latency 측정
```bash
curl -w "TTFB: %{time_starttransfer}s, Total: %{time_total}s\n" \
     -o /dev/null -s https://example.com
```

### 4.8 Repeating
```bash
for i in $(seq 1 100); do
    curl -s -o /dev/null -w "%{http_code} %{time_total}\n" https://api/health
done
```

---

## 5. curl-format.txt 템플릿

```
time_namelookup:    %{time_namelookup}s
time_connect:       %{time_connect}s
time_appconnect:    %{time_appconnect}s
time_pretransfer:   %{time_pretransfer}s
time_redirect:      %{time_redirect}s
time_starttransfer: %{time_starttransfer}s
                    ----------
time_total:         %{time_total}s
```

```bash
curl -w "@curl-format.txt" -o /dev/null -s https://example.com
```

---

## 6. curl 와 비슷 / 대체

### wget
```bash
wget https://example.com/file       # 다운로드 친화
wget -r -np https://example.com     # 재귀 (mirror)
```

### nc (netcat) — 원시 TCP
```bash
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | nc example.com 80
```

### openssl s_client — 원시 TLS
```bash
openssl s_client -connect example.com:443 -crlf
GET / HTTP/1.1
Host: example.com

```

---

## 7. 함정

### 함정 1 — -d 의 URL encoding
`curl -d 'name=alice & bob'` — `&` 가 새 param. `--data-urlencode` 사용.

### 함정 2 — -X 의 자동 method
`-X POST` 만으로 — 빈 body. `-d ''` 또는 `-d @body`.

### 함정 3 — Redirect 의 method 보존
POST → 301 → GET 가 됨. `-L --post301 --post302` 강제.

### 함정 4 — 옛 TLS 의 자동 disable
TLS 1.3 만 가능 — 옛 server fail. `--tls-max` 명시.

### 함정 5 — Cookie 의 영속
같은 jar — 누적. `-c /dev/null -b cookies.txt` 분리.

### 함정 6 — HTTP/2 의 multiplexing
한 curl — 한 stream. ParallelCurl / vegeta 사용.

### 함정 7 — `-k` 의 보안
TLS 검증 무시 — MITM. 개발만.

---

## 8. 학습 자료

- "curl & libcurl" — Daniel Stenberg
- `man curl`
- httpie.io docs
- "everything-curl" (Stenberg)

---

## 9. 관련

- [[tools]] — Hub
- [[tcpdump-wireshark]] — 더 깊은 디버깅
- [[../http/methods/methods]] — HTTP method
- [[../tls-ssl/tls-ssl]] — TLS handshake
