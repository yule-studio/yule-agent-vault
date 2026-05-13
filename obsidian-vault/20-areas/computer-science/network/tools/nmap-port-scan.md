---
title: "nmap — 포트 스캔 / 발견"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T09:15:00+09:00
tags:
  - network
  - tools
  - nmap
  - security
---

# nmap — 포트 스캔 / 발견

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 스캔 / OS detect / 스크립트 |

**[[tools|↑ 도구 hub]]** · **[[../network|↑↑ network hub]]**

---

## 1. nmap — Network Mapper

### 1997, Gordon Lyon (Fyodor)

### 정의
- 네트워크 / 호스트 / 서비스 발견
- 포트 스캔
- OS / 버전 detection
- 보안 감사

---

## 2. 기본 스캔

```bash
# 단일 호스트 (TCP SYN scan, top 1000 port)
nmap example.com

# 자세히
nmap -v example.com

# IP / CIDR
nmap 192.168.1.0/24
nmap 1.2.3.4-100
nmap -iL hosts.txt              # 파일에서
```

---

## 3. 스캔 타입

### TCP SYN scan (-sS) — 기본 (root)
- 반-open scan (SYN, SYN-ACK 받으면 RST)
- 빠르고 stealth

### TCP Connect scan (-sT) — non-root
- full connect
- 로그에 남음
- 일반 사용자

### UDP scan (-sU)
- 느림 (응답 없으면 timeout)
- DNS / NTP / SNMP

### Ping scan (-sn / -sP)
- Port scan 없이 — 활성 호스트만
- ICMP / ARP

### List scan (-sL)
- 스캔 없이 — IP 목록만

### Stealth scan
- `-sF` (FIN), `-sN` (NULL), `-sX` (Xmas)
- 일부 방화벽 우회

### TCP ACK scan (-sA)
- 방화벽 rule 발견

---

## 4. 포트 옵션

```bash
nmap -p 80,443 example.com           # 특정 port
nmap -p 1-1000 example.com           # range
nmap -p- example.com                 # 모든 port (1-65535)
nmap -p T:80,U:53 example.com        # TCP + UDP
nmap --top-ports 10 example.com      # 인기 10
nmap -F example.com                  # Fast (100 port)
```

---

## 5. 서비스 / 버전 detection

```bash
nmap -sV example.com
# 결과
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.9p1
80/tcp   open  http     nginx 1.22
443/tcp  open  https    nginx 1.22
```

### 옵션
```bash
nmap -sV --version-intensity 9 example.com   # 더 자세히
nmap -sV --version-light example.com          # 빠르게
```

---

## 6. OS detection

```bash
nmap -O example.com
nmap -A example.com                  # 모든 (OS + version + script + traceroute)
```

### 결과
```
OS details: Linux 5.4 - 5.15
Network Distance: 8 hops
```

### 정확도
- 일반 — 80%+
- 방화벽 / NAT 뒤 — 부정확

---

## 7. Speed / timing

```bash
nmap -T0 example.com    # paranoid (느림, stealth)
nmap -T1 example.com    # sneaky
nmap -T2 example.com    # polite
nmap -T3 example.com    # normal (기본)
nmap -T4 example.com    # aggressive
nmap -T5 example.com    # insane (빠름, 부정확)
```

### 부하 제어
```bash
nmap --min-rate 100 --max-rate 1000 example.com
nmap --max-retries 1 example.com
```

---

## 8. NSE — Nmap Scripting Engine

### 정의
- Lua 기반 스크립트
- 600+ 내장

### 사용
```bash
nmap --script default example.com           # 기본 스크립트
nmap --script vuln example.com               # 취약점
nmap --script http-* example.com             # HTTP 관련 모두

# 특정 스크립트
nmap --script ssl-enum-ciphers -p 443 example.com
nmap --script http-headers -p 80 example.com
nmap --script dns-brute example.com
nmap --script smb-vuln-* -p 445 192.168.1.0/24
```

### 카테고리
- auth, broadcast, brute, default, discovery
- dos, exploit, external, fuzzer, intrusive
- malware, safe, version, vuln

### 인기 스크립트
| 스크립트 | 용도 |
| --- | --- |
| ssl-enum-ciphers | TLS cipher 검사 |
| http-title | 페이지 제목 |
| http-headers | 응답 헤더 |
| http-methods | 허용 method |
| smb-os-discovery | Windows SMB |
| ssh-hostkey | SSH 키 |
| dns-zone-transfer | DNS AXFR |
| ftp-anon | 익명 FTP |

---

## 9. 출력 포맷

```bash
nmap -oN output.txt example.com         # normal
nmap -oX output.xml example.com         # XML
nmap -oG output.grep example.com        # grepable
nmap -oA output example.com             # 모든 포맷
```

---

## 10. 흔한 시나리오

### 10.1 보안 감사
```bash
# 외부 노출 포트
sudo nmap -sS -p- -T4 your-server.com

# 모든 정보
sudo nmap -A -T4 your-server.com
```

### 10.2 내부 망 발견
```bash
sudo nmap -sn 192.168.1.0/24        # 활성 호스트
sudo nmap -sV 192.168.1.0/24        # 서비스
```

### 10.3 TLS 강도 확인
```bash
nmap --script ssl-enum-ciphers -p 443 example.com
# 옛 cipher / weak DH / SSLv3 등 발견
```

### 10.4 SSH 키
```bash
nmap --script ssh-hostkey example.com
```

### 10.5 SMB 취약점
```bash
sudo nmap --script smb-vuln-* -p 445 192.168.1.10
# EternalBlue (MS17-010) 등
```

### 10.6 방화벽 우회 시도
```bash
nmap -sA example.com           # ACK scan
nmap -f example.com            # fragment
nmap --mtu 32 example.com      # 작은 MTU
nmap -D 1.2.3.4 example.com    # decoy
```

---

## 11. 빠른 대체 — masscan / rustscan

### masscan
- 매우 빠름 (분 단위로 전체 인터넷)
- Robert Graham (2014)

```bash
sudo masscan -p1-65535 1.2.3.0/24 --rate 10000
```

### rustscan
- Rust 작성
- 빠른 port discovery → nmap 으로 detail

```bash
rustscan -a example.com -- -A -sV
```

---

## 12. 법 / 윤리

### 허가 없는 스캔 = 불법
- 미국 — CFAA
- 한국 — 정보통신망법

### 자기 자산 / 허가된 자산만
- 회사 — pentesting 계약
- 본인 server
- HackTheBox / TryHackMe (랩)

### Cloud
- AWS — 자기 인스턴스 OK, 다른 사용자 X
- GCP — 사전 신고 권장 (큰 스캔)

---

## 13. 함정

### 함정 1 — Stealth ≠ 안 보임
모던 IDS — SYN scan 도 detect.

### 함정 2 — UDP scan 의 false positive
응답 없음 = open|filtered. 정확하지 않음.

### 함정 3 — Top port 의 누락
서비스 가 임의 port — `-p-` 필요.

### 함정 4 — 큰 망 스캔 시간
/16 + `-p-` = 며칠. masscan 권장.

### 함정 5 — Rate limit / IDS
빠른 스캔 — block. `-T2` 권장 (운영).

### 함정 6 — DNS 의 자동 해석
`-n` (no DNS) — 빠름.

### 함정 7 — Cloud / WAF 가 가짜 응답
모든 port 가 open 으로 보일 수 있음. Cloud 의 보안 그룹 확인.

---

## 14. 학습 자료

- "Nmap Network Scanning" (Lyon, 무료 온라인)
- nmap.org
- `man nmap`
- HackTheBox / TryHackMe

---

## 15. 관련

- [[tools]] — Hub
- [[tcpdump-wireshark]] — 결과 분석
- [[../tls-ssl/cipher-suites]] — ssl-enum-ciphers
