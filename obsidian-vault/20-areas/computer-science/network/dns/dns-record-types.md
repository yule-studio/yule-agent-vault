---
title: "DNS Record Types"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:30:00+09:00
tags:
  - network
  - dns
  - records
---

# DNS Record Types

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | A/AAAA/CNAME/MX/TXT/NS/SOA/SRV/PTR/CAA/HTTPS |

**[[dns|↑ DNS]]** · **[[../network|↑↑ network hub]]**

---

## 1. 주요 Record 한눈

| Type | 용도 | 예 |
| --- | --- | --- |
| **A** | 도메인 → IPv4 | `1.2.3.4` |
| **AAAA** | 도메인 → IPv6 | `2001:db8::1` |
| **CNAME** | 도메인 → 도메인 별칭 | `www.example.com → example.com` |
| **MX** | 메일 서버 | `mail.example.com priority 10` |
| **TXT** | 임의 텍스트 (인증) | SPF, DKIM, DMARC |
| **NS** | 도메인의 권한 서버 | `ns1.example.com` |
| **SOA** | Zone 메타 | serial / TTL |
| **SRV** | 서비스 (port 포함) | `_xmpp-server._tcp` |
| **PTR** | IP → 도메인 (역) | `4.3.2.1.in-addr.arpa` |
| **CAA** | CA 인증 정책 | `0 issue "letsencrypt.org"` |
| **HTTPS** / **SVCB** | HTTP/3 발견 | `alpn=h3,h2` |
| **DNSKEY / RRSIG / DS** | DNSSEC | 서명 / 키 |

---

## 2. A — IPv4

```
example.com.    300  IN  A  1.2.3.4
www.example.com. 300  IN  A  1.2.3.4
```

### 형식
```
<name> <TTL> <class> <type> <data>
```

- **name** — 도메인
- **TTL** — 캐시 시간 (초)
- **class** — IN (Internet) 가 99%
- **type** — A
- **data** — IPv4

### Multiple A — Load Balancing
```
example.com.   IN  A  1.2.3.4
example.com.   IN  A  5.6.7.8
example.com.   IN  A  9.10.11.12
```

→ Round-robin DNS. 클라가 받은 순서로 시도.

### Geo / Latency DNS
- 클라 위치 별 다른 응답
- AWS Route 53 Geolocation / Latency
- Cloudflare Geo-Steering

---

## 3. AAAA — IPv6

```
example.com.   IN  AAAA  2001:db8::1
```

→ A 의 IPv6 버전.

### Dual-stack
```
example.com.   IN  A     1.2.3.4
example.com.   IN  AAAA  2001:db8::1
```

→ Happy Eyeballs — IPv6 우선 시도.

자세히 → [[../ip/ipv4-vs-ipv6]]

---

## 4. CNAME — Canonical Name

```
www.example.com.   IN  CNAME  example.com.
api.example.com.   IN  CNAME  loadbalancer.aws.com.
```

→ 다른 도메인 의 별칭.

### 제약
- **Root domain 에 CNAME X** (RFC 1034)
- `example.com IN CNAME ...` 가 위반

### 우회 — ALIAS / ANAME
- AWS Route 53 — ALIAS
- Cloudflare — CNAME Flattening
- DNSimple — ALIAS

→ DNS 응답 시 자동 A / AAAA 로 변환.

### CNAME chain 제한
- 8 단계 (RFC) — 무한 loop 방지
- 실제 — 1-2 hop 권장

---

## 5. MX — Mail Exchange

```
example.com.   IN  MX  10  mail1.example.com.
example.com.   IN  MX  20  mail2.example.com.
```

### Priority
- 낮을수록 우선
- 같은 priority → load balancing

### MX 의 target
- A / AAAA 가 있어야 (CNAME X)
- 일부 mail server 는 받지만 RFC 위반

자세히 → [[../mail/mail]]

---

## 6. TXT — Text

```
example.com.   IN  TXT  "v=spf1 include:_spf.google.com ~all"
example.com.   IN  TXT  "google-site-verification=..."
_dmarc.example.com.   IN  TXT  "v=DMARC1; p=reject; ..."
```

### 사용
- **SPF** (Sender Policy Framework) — 메일 인증
- **DKIM** — 메일 서명
- **DMARC** — 메일 정책
- **Domain verification** (Google / GitHub / Slack)
- **Let's Encrypt** DNS-01 challenge

자세히 → [[../mail/spf-dkim-dmarc]]

---

## 7. NS — Name Server

```
example.com.   IN  NS  ns1.example.com.
example.com.   IN  NS  ns2.example.com.
```

→ 이 도메인의 권한 보유 서버.

### Glue Record
```
example.com.   IN  NS  ns1.example.com.
ns1.example.com.   IN  A   1.2.3.10        ← Glue
```

→ 같은 도메인 안의 NS 이면 IP 도 함께.

---

## 8. SOA — Start of Authority

```
example.com.   IN  SOA  ns1.example.com. admin.example.com. (
    2024051401   ; serial — 매 변경 시 증가 (YYYYMMDDNN)
    3600         ; refresh — secondary 가 검증 간격
    1800         ; retry — refresh 실패 시 재시도
    1209600      ; expire — 14 일, secondary 가 동기 못 하면 폐기
    300          ; minimum TTL — negative cache
)
```

### 의미
- Zone 의 권한 시작
- Primary NS + admin 이메일 (`.` 가 `@` 의 자리)
- Serial 로 변경 추적

---

## 9. SRV — Service

```
_xmpp-server._tcp.example.com.   IN  SRV  10 0 5269 xmpp.example.com.
_minecraft._tcp.example.com.     IN  SRV  10 0 25565 mc.example.com.
```

### 형식
```
_<service>._<proto>.<domain>   SRV  <priority> <weight> <port> <target>
```

### 사용
- XMPP, SIP, LDAP, Kerberos
- Minecraft, Discord 등 일부
- Active Directory

### 한계
- HTTP / HTTPS 에 사용 X (브라우저 무시)
- 모던 — HTTPS / SVCB record 가 대체

---

## 10. PTR — Reverse DNS

```
4.3.2.1.in-addr.arpa.   IN  PTR  example.com.
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa. IN PTR example.com.
```

### 사용
- IP → 도메인 (역방향)
- **메일 서버** — 강한 검증 (SPF + PTR + DKIM)
- 로그 분석

### 설정
- 일반 도메인 owner 와 다른 운영자 (ISP / Cloud)
- AWS — Route 53 의 reverse zone
- Hetzner / Linode — UI 에서 설정

---

## 11. CAA — Certificate Authority Authorization

```
example.com.   IN  CAA  0 issue "letsencrypt.org"
example.com.   IN  CAA  0 issuewild ";"        ; wildcard 금지
example.com.   IN  CAA  0 iodef "mailto:security@example.com"
```

### 의미
- 어느 CA 가 이 도메인의 cert 발급 가능
- CA 가 발급 전 CAA 검증 의무 (RFC 8659)
- 잘못된 CA 가 cert 발급 방지

---

## 12. HTTPS / SVCB — 모던 (RFC 9460)

```
example.com.   IN  HTTPS  1 . alpn="h3,h2" port=443 ipv4hint=1.2.3.4 ipv6hint=2001:db8::1
```

### 의미
- 한 record 에 — ALPN (HTTP/3 발견) + IP hints + port
- Alt-Svc 의 DNS 버전
- 첫 방문도 HTTP/3 사용 가능

### 도입
- Apple iOS 14+, macOS 11+ (2020)
- Chrome / Cloudflare 점진

자세히 → [[../http/versions/version-comparison]]

---

## 13. DNSSEC Records

자세히 → [[dnssec]]

### DNSKEY
- Zone 의 공개키

### RRSIG
- Resource Record 의 서명

### DS — Delegation Signer
- 부모 zone 에 자식 zone 의 키 hash
- 신뢰 체인

### NSEC / NSEC3
- "이 도메인은 없다" 의 증명 (negative response 서명)

---

## 14. 기타 / 옛 record

| Type | 용도 |
| --- | --- |
| HINFO | 호스트 정보 (사장) |
| LOC | 위치 좌표 |
| NAPTR | URI 매핑 (SIP) |
| SSHFP | SSH 공개키 fingerprint |
| TLSA | DANE — TLS cert pinning |
| URI | URI 매핑 |
| LOC | GPS 좌표 |
| OPENPGPKEY | PGP 키 |
| SMIMEA | S/MIME |

---

## 15. Record 의 TTL

```
example.com.   300   IN  A  1.2.3.4
                ↑ 5 분 캐시
```

### 권장
- 자주 변하는 — 60-300 (1-5 분)
- 안정 — 3600-86400 (1-24 시간)
- Maintenance 전 — 짧게 (60), 후 길게

---

## 16. 도구로 조회

```bash
# 특정 type
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com TXT
dig example.com NS
dig example.com CAA
dig example.com HTTPS

# 모든 record (옛 도구는 안 됨 — 일부 NS 거부)
dig example.com ANY

# 짧게
dig +short example.com
```

---

## 17. 함정

### 함정 1 — Root CNAME
RFC 위반. ALIAS 사용.

### 함정 2 — MX 의 CNAME
RFC 위반. A / AAAA 만.

### 함정 3 — TXT 의 길이 (255 byte)
한 string 한계. 여러 string 으로 ("...string1" "string2") 또는 multi-line.

### 함정 4 — CAA 누락
악의적 CA 가 cert 발급 가능. CAA 권장.

### 함정 5 — Reverse PTR 누락
메일 발송 시 차단. ISP 에 설정.

### 함정 6 — Geo DNS 의 짧은 TTL
계속 조회 → DNS 부하.

---

## 18. 학습 자료

- **RFC 1035** (기본 record)
- **RFC 9460** (HTTPS / SVCB)
- **RFC 8659** (CAA)
- IANA DNS Parameters
- Cloudflare DNS docs

---

## 19. 관련

- [[dns]] — DNS hub
- [[dns-hierarchy-resolution]] — NS / SOA 의 역할
- [[dnssec]] — DNSSEC record
- [[../mail/spf-dkim-dmarc]] — TXT 활용
