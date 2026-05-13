---
title: "SPF / DKIM / DMARC — 메일 인증"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T04:15:00+09:00
tags:
  - network
  - mail
  - spf
  - dkim
  - dmarc
  - security
---

# SPF / DKIM / DMARC — 메일 인증

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | SPF / DKIM / DMARC / BIMI |

**[[mail|↑ Mail]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **표준** | SPF (RFC 7208), DKIM (RFC 6376), DMARC (RFC 7489) |
| **저장** | DNS TXT |
| **목적** | 메일 위조 / spoofing 방지 |

---

## 1. 한 줄 정의

도메인의 **메일 발송 권한 / 무결성 / 정책** 을 DNS TXT 로 선언. 받는 서버가 검증해 spam / 위조 차단.

---

## 2. 세 가지 — 한눈

| 기술 | 검증 | 어떻게 |
| --- | --- | --- |
| **SPF** | 발송 IP 가 허용된 IP 인가 | DNS TXT 의 IP 리스트 |
| **DKIM** | 메일이 변조 안 됐는가 | 디지털 서명 (DNS 의 공개키) |
| **DMARC** | SPF / DKIM 실패 시 정책 | reject / quarantine / none |

---

## 3. SPF — Sender Policy Framework

### 정의
- 도메인에서 메일을 보낼 수 있는 **IP 주소 리스트** 를 DNS 에 선언

### DNS record
```
example.com.   TXT  "v=spf1 ip4:1.2.3.0/24 include:_spf.google.com ~all"
```

### 메커니즘
| 메커니즘 | 의미 |
| --- | --- |
| `ip4:` / `ip6:` | IP / CIDR 허용 |
| `a` | 도메인의 A record IP 허용 |
| `mx` | MX record 의 IP 허용 |
| `include:` | 다른 도메인의 SPF 포함 |
| `redirect=` | 다른 도메인의 SPF 로 위임 |
| `exists:` | 도메인 존재 시 매치 |

### Qualifier
| | 의미 | 동작 |
| --- | --- | --- |
| `+` | Pass | 허용 (기본) |
| `-` | Hard fail | 거부 |
| `~` | Soft fail | 의심 (spam 폴더) |
| `?` | Neutral | 모름 |

### `all` 의 의미
```
~all     ; 위 매치 안 된 것 — soft fail
-all     ; hard fail (엄격)
?all     ; neutral (테스트)
```

### 검증
받는 서버:
1. MAIL FROM 의 도메인 — 예 `alice@example.com`
2. `example.com` 의 SPF TXT 조회
3. 발송 IP 가 매치되는지

### 한계
- **Forwarding 실패** — 메일 forwarding (mailing list) 시 IP 변경 → SPF 실패
- **MAIL FROM 만 검증** — From 헤더 (실제 표시) 와 다를 수 있음
- **DNS lookup 10 개 한계** — include 깊으면 PermError

자세히 → [[../dns/dns-record-types]]

---

## 4. DKIM — DomainKeys Identified Mail

### 정의
- 발송 서버가 메일에 **디지털 서명**
- 받는 서버가 DNS 의 공개키로 검증
- 본문 / 헤더 무결성

### DNS record
```
selector1._domainkey.example.com.   TXT  "v=DKIM1; k=rsa; p=MIGfMA0..."
```

### 메일 헤더
```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
  d=example.com; s=selector1; t=1700000000;
  h=From:To:Subject:Date; bh=...; b=...
```

| 필드 | 의미 |
| --- | --- |
| `d` | 도메인 |
| `s` | Selector (selector1._domainkey) |
| `h` | 서명에 포함된 헤더 |
| `bh` | 본문의 해시 |
| `b` | 서명 |

### 검증
1. 받는 서버가 `s + d` 로 DNS 조회 → 공개키
2. 메일의 헤더 + 본문 정규화
3. 공개키로 서명 검증

### Canonicalization
- **simple** — 거의 그대로
- **relaxed** — 공백 정규화 (forwarding 에 강함)

### 효과
- 본문 변경 — bh 불일치
- 헤더 변조 — b 불일치
- → spam 신호

### Forwarding 강함
- IP 가 바뀌어도 서명은 유지
- SPF 약점 보완

---

## 5. DMARC — Domain-based Message Authentication

### 정의
- SPF / DKIM 의 **결과를 종합** 해 어떻게 처리할지 정책
- 실패 시 reject / quarantine / none

### DNS record
```
_dmarc.example.com.   TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com; pct=100; adkim=s; aspf=s"
```

### 정책 (`p=`)
| 정책 | 의미 |
| --- | --- |
| `none` | 모니터만 (보고서) |
| `quarantine` | spam 폴더로 |
| `reject` | 거부 (drop) |

### Alignment
- `adkim` / `aspf` — strict / relaxed
- From 헤더의 도메인이 SPF / DKIM 의 도메인과 일치해야

### 보고서 (`rua`, `ruf`)
```
rua=mailto:aggregate@example.com   ; 집계 보고
ruf=mailto:forensic@example.com    ; 개별 보고
```

→ XML / 우편함으로 받음. 도구 — DMARC Analyzer, dmarcian.

### Percent (`pct=`)
- 부분 적용 — 점진 도입
- 처음 — pct=10 → 50 → 100

### 도입 단계
```
1. p=none, pct=100, rua= 보고 받기
   → 도메인의 메일 흐름 파악
2. p=quarantine, pct=10 → 50 → 100
   → spam 폴더 점진
3. p=reject, pct=10 → 100
   → 차단
```

---

## 6. BIMI — Brand Indicators (RFC draft)

### 정의
- 메일 클라이언트에 **브랜드 로고** 표시
- DMARC `p=reject` 필요

### DNS record
```
default._bimi.example.com.   TXT  "v=BIMI1; l=https://example.com/logo.svg; a=https://example.com/vmc.pem"
```

| 필드 | 의미 |
| --- | --- |
| `l` | 로고 SVG URL |
| `a` | VMC (Verified Mark Certificate) — 상표권 검증 |

### 효과
- Gmail / Apple Mail — 로고 표시
- 브랜드 / 신뢰

### 한계
- VMC 비쌈 ($1000+/년)
- DMARC reject 필수

---

## 7. ARC — Authenticated Received Chain (RFC 8617)

### 문제
- Mailing list 등 forwarding 시 SPF / DMARC 깨짐
- DKIM 도 본문 수정 시 깨짐

### ARC
- 중간 서버가 검증 결과를 **새 서명** 으로 보존
- 다음 서버 — ARC chain 검증

### 헤더
```
ARC-Authentication-Results: i=1; ...
ARC-Message-Signature: i=1; ...
ARC-Seal: i=1; ...
```

→ 큰 mailing list / Google Groups 에 도움.

---

## 8. 흐름 — 전체

```
Sender (alice@example.com)
  → SMTP → mail.example.com

Send:
  - SPF — mail.example.com 가 example.com 의 허용 IP
  - DKIM — example.com 의 selector1 키로 서명

받는 서버 (mail.recipient.com):
  1. SPF 검증 — MAIL FROM 도메인 + 발송 IP
  2. DKIM 검증 — DNS 조회 + 서명
  3. DMARC 검증
     - From 헤더의 도메인 alignment
     - SPF / DKIM 중 하나라도 pass + aligned → DMARC pass
     - 둘 다 fail → DMARC p= 정책 적용
  4. 결과 — Authentication-Results 헤더 추가
  5. spam 점수 / 폴더 분류
```

---

## 9. Authentication-Results 헤더

```
Authentication-Results: mx.recipient.com;
  spf=pass (mx.recipient.com: domain of alice@example.com designates 1.2.3.4 as permitted sender) smtp.mailfrom=alice@example.com;
  dkim=pass header.i=@example.com header.s=selector1 header.b=...;
  dmarc=pass (p=reject sp=reject dis=none) header.from=example.com
```

→ 받는 서버가 검증 결과 기록.

---

## 10. 도구 / 검증

### DNS 조회
```bash
dig example.com TXT                    ; SPF
dig selector1._domainkey.example.com TXT  ; DKIM
dig _dmarc.example.com TXT             ; DMARC
dig default._bimi.example.com TXT      ; BIMI
```

### 온라인 도구
- **mxtoolbox.com** — SPF / DKIM / DMARC 검증
- **mail-tester.com** — 메일 보내고 점수 받기
- **dmarcian.com** — DMARC 분석 / 보고
- **easydmarc.com** — DMARC dashboard

### Postmark
- DMARC Digest — 무료 DMARC 보고서 요약

---

## 11. 모던 메일 발송 체크리스트

```
[ ] SPF — v=spf1 ... ~all (도입 시) → -all (안정 시)
[ ] DKIM — 2048-bit 키
[ ] DMARC — p=none (관찰) → p=quarantine → p=reject
[ ] Reverse DNS (PTR) — 발송 IP
[ ] MX record — A / AAAA (CNAME X)
[ ] TLS — STARTTLS / MTA-STS
[ ] IP reputation — Sender Score, Talos
[ ] Bounce 관리 — 5xx 즉시 제거
[ ] Unsubscribe — List-Unsubscribe 헤더
[ ] BIMI — 브랜드 (선택)
```

---

## 12. 함정

### 함정 1 — SPF 10 lookup 한계
include: 의 깊이 합산 10 개 한계. PermError.
"flat SPF" 도구 — SPF 평탄화.

### 함정 2 — DKIM key rotation X
키 회전 안 함 — 유출 시 영구. 1 년 마다 rotate 권장.

### 함정 3 — DMARC alignment 의 strict / relaxed
- relaxed — example.com 과 mail.example.com 매치
- strict — 정확히 같아야

### 함정 4 — BIMI 의 VMC 비용
$1000+/년. 대기업 / 브랜드만.

### 함정 5 — DMARC reject 도입 너무 빠름
정상 메일도 reject. p=none → pct 점진.

### 함정 6 — Forwarding 의 SPF/DKIM 깨짐
ARC 도입 / SRS (Sender Rewriting Scheme).

### 함정 7 — DKIM signature 의 본문 수정
List footer 추가 → 본문 해시 변경 → DKIM 실패. l= tag (length) 또는 ARC.

---

## 13. 학습 자료

- **RFC 7208** (SPF)
- **RFC 6376** (DKIM)
- **RFC 7489** (DMARC)
- **RFC 8617** (ARC)
- **RFC 8616** (BIMI draft)
- "DMARC.org" / "M3AAWG Sender Best Practices"

---

## 14. 관련

- [[mail]] — Mail hub
- [[smtp]] — 송신
- [[../dns/dns-record-types]] — TXT
- [[../dns/dnssec]] — DNS 무결성
- [[../tls-ssl/tls-ssl]] — TLS
