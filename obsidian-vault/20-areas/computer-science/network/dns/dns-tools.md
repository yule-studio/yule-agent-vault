---
title: "DNS 도구 — dig / nslookup / host / dnsperf"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:50:00+09:00
tags:
  - network
  - dns
  - tools
  - dig
---

# DNS 도구 — dig / nslookup / host / dnsperf

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | dig / nslookup / host + 분석 |

**[[dns|↑ DNS]]** · **[[../network|↑↑ network hub]]**

---

## 1. dig — 가장 강력

### 기본
```bash
dig example.com
```

### 출력 해석
```
;; QUESTION SECTION:
;example.com.                  IN      A

;; ANSWER SECTION:
example.com.           300     IN      A       1.2.3.4

;; AUTHORITY SECTION:
example.com.           172800  IN      NS      ns1.example.com.

;; Query time: 35 msec
;; SERVER: 1.1.1.1#53(1.1.1.1)
;; WHEN: Tue May 14 12:00:00 KST 2026
;; MSG SIZE  rcvd: 60
```

### 옵션

```bash
# 짧게
dig +short example.com

# 특정 type
dig example.com A
dig example.com AAAA
dig example.com MX
dig example.com TXT
dig example.com NS
dig example.com CAA
dig example.com HTTPS
dig example.com SOA

# 특정 NS 에 질문
dig @8.8.8.8 example.com
dig @ns1.example.com example.com

# Recursive 비활성
dig +norecurse @ns1.example.com example.com

# Trace (Root 부터)
dig +trace example.com

# DNSSEC
dig +dnssec example.com
dig +cd example.com         # Checking Disabled (DNSSEC 검증 X)

# TCP 강제
dig +tcp example.com

# 모든 정보
dig +all +multiline example.com

# 통계 숨김
dig +nostats example.com
```

### 출력 형식
```bash
# JSON 비슷
dig +noall +answer +short example.com
# 1.2.3.4
```

---

## 2. nslookup — 모든 OS

### 기본
```bash
nslookup example.com
# Server:		1.1.1.1
# Address:	1.1.1.1#53
#
# Non-authoritative answer:
# Name:	example.com
# Address: 1.2.3.4
```

### 특정 type
```bash
nslookup -type=MX example.com
nslookup -type=TXT example.com
nslookup -type=SOA example.com
```

### 특정 NS
```bash
nslookup example.com 8.8.8.8
```

### Interactive
```
$ nslookup
> set type=mx
> example.com
> set type=ns
> google.com
> exit
```

---

## 3. host — 짧고 간단

```bash
host example.com
# example.com has address 1.2.3.4
# example.com has IPv6 address 2001:db8::1
# example.com mail is handled by 10 mail.example.com.

# 특정 type
host -t MX example.com
host -t TXT example.com

# Reverse
host 1.2.3.4
# 4.3.2.1.in-addr.arpa domain name pointer example.com.
```

---

## 4. drill — Linux 의 대체

dig 와 비슷 — `ldns-utils` 패키지:
```bash
drill example.com
drill -D example.com    # DNSSEC
```

---

## 5. kdig (knot-dnsutils)

dig 의 모던 대체. DoT / DoH 지원:
```bash
kdig +tls @1.1.1.1 example.com
kdig +https @1.1.1.1 example.com
```

---

## 6. dnsperf — 성능 측정

```bash
# query 리스트
echo "example.com A" > queries.txt
echo "www.example.com A" >> queries.txt

# 부하 측정
dnsperf -d queries.txt -s 1.1.1.1 -l 60 -c 10
# -l: 60 초
# -c: 10 동시 client
```

### 결과
```
QPS (Queries per Second)
Avg latency
Failed queries
```

---

## 7. resperf — 점진 부하

```bash
resperf -d queries.txt -s 1.1.1.1 -m 1000 -t 30
# -m: max QPS
# -t: 30 초
```

→ DNS server 의 성능 한계 찾기.

---

## 8. delv — DNSSEC validation

```bash
delv +rtrace example.com
# DNSSEC chain 따라가며 검증
```

---

## 9. whois — 도메인 등록 정보

```bash
whois example.com
```

### 정보
- 등록자 (privacy 마스킹 가능)
- 등록일 / 만료일
- Name server
- Registrar

---

## 10. mtr 의 DNS

```bash
mtr --report --report-cycles 10 example.com
```

→ DNS 조회 + 경로 추적 결합.

---

## 11. tcpdump — 패킷 캡처

```bash
# UDP 53 만
sudo tcpdump -i en0 -nn 'udp port 53'

# 자세히
sudo tcpdump -i en0 -nn -v 'udp port 53 or tcp port 53'

# DNSSEC (TCP 53 / EDNS0)
sudo tcpdump -i en0 -nn -X 'udp port 53'
```

---

## 12. Wireshark

### Display filter
```
dns
dns.qry.name == "example.com"
dns.flags.response == 1
dns.qry.type == 1   (A)
dns.qry.type == 28  (AAAA)
```

---

## 13. 온라인 도구

- **dig.dns.us-east-1.amazonaws.com** — AWS 의 web dig
- **mxtoolbox.com** — MX 와 다양한 record
- **dnschecker.org** — 전 세계 propagation
- **crt.sh** — Certificate Transparency 의 도메인 조회
- **bgp.he.net** — Hurricane Electric BGP / DNS

---

## 14. 흔한 시나리오 — 디버깅

### 14.1 도메인이 안 떠
```bash
dig example.com +trace
# 어느 단계에서 실패?
# - Root → .com → example.com 의 NS 까지 잘 되는지
```

### 14.2 메일 안 옴
```bash
dig example.com MX
dig mail.example.com A
dig _domainkey.example.com TXT     # DKIM
dig _dmarc.example.com TXT
host -t TXT example.com            # SPF
```

### 14.3 HTTPS Cert 발급 안 됨
```bash
dig example.com CAA
# CAA 가 letsencrypt.org 허용?
```

### 14.4 Reverse DNS (메일 / 로그)
```bash
host 1.2.3.4
# 또는
dig -x 1.2.3.4
```

### 14.5 새 NS 적용 확인
```bash
dig @8.8.8.8 example.com NS
dig @1.1.1.1 example.com NS
dig @{원본 NS} example.com NS
# 일관 — propagation OK
# 다름 — TTL 따라 점진
```

---

## 15. 함정

### 함정 1 — dig 의 ANY query
일부 NS 가 거부 (DoS amplification 방어). 명시 type 사용.

### 함정 2 — Cache 의 영향
`+trace` 사용 시만 root 부터 조회. 일반은 cache 따름.

### 함정 3 — nslookup 의 옛 동작
- `Non-authoritative answer` — cache 응답. 의미 큰 X.
- 일부 옛 nslookup — TCP 옵션 없음.

### 함정 4 — Time / Date 의 동기
DNSSEC 검증 — 시계 정확해야.

### 함정 5 — IPv6 만 응답
일부 도메인 — AAAA 만, A 없음. dig -t A / dig -t AAAA 둘 다.

---

## 16. 학습 자료

- man dig (`man 1 dig`)
- "BIND 9 Administrator Reference Manual"
- DNS 다국적 도구 — dnsperf / resperf docs

---

## 17. 관련

- [[dns]] — DNS hub
- [[dns-record-types]] — 어떤 record 가 있는지
- [[../tools/tools]] — 네트워크 도구 hub
