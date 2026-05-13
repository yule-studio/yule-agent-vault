---
title: "SMTP — Simple Mail Transfer Protocol"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T04:00:00+09:00
tags:
  - network
  - mail
  - smtp
---

# SMTP — Simple Mail Transfer Protocol

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RFC 5321 / 명령어 / STARTTLS / SUBMISSION |

**[[mail|↑ Mail]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **포트** | 25 (server-to-server), 587 (submission), 465 (SMTPS) |
| **표준** | RFC 5321 (2008) — 옛 RFC 821 (1982) |
| **암호화** | STARTTLS (25/587), implicit TLS (465) |

---

## 1. 한 줄 정의

**메일 송신** 의 표준 프로토콜. 텍스트 기반 (request-response). 1982 부터 인터넷의
메일 운반.

---

## 2. 3 가지 포트

### 25 — Server-to-Server (MTA)
- 옛 표준
- ISP 가 outbound 차단 (Spam 방지)
- 클라이언트가 사용 X

### 587 — Submission
- 클라이언트 → SMTP server (RFC 6409)
- AUTH 필수
- STARTTLS

### 465 — SMTPS (Implicit TLS)
- 옛 — 사장 → 부활 (RFC 8314, 2018)
- TLS 시작부터
- Submission 의 TLS 버전

---

## 3. SMTP 명령

```
HELO / EHLO        — 인사 (서버 식별)
MAIL FROM:         — 발신자
RCPT TO:           — 수신자 (여러 번 OK)
DATA               — 본문 시작
.                  — 본문 끝 (단독 . 라인)
RSET               — 리셋
NOOP               — 핑
QUIT               — 종료

AUTH               — 인증 (587)
STARTTLS           — TLS 시작
```

### EHLO (Extended HELLO)
- ESMTP 확장 — 8BITMIME, SIZE, AUTH, STARTTLS 등 광고

---

## 4. 흐름 — 전체

```
Client connect: TCP 25
Server: 220 mail.example.com ESMTP Postfix

Client: EHLO myhost.com
Server: 250-mail.example.com
        250-STARTTLS
        250-AUTH PLAIN LOGIN
        250-SIZE 10240000
        250 8BITMIME

Client: STARTTLS
Server: 220 Go ahead

[TLS handshake]

Client: EHLO myhost.com    ← TLS 위에서 다시
Server: 250-...
        250 ...

Client: AUTH PLAIN <base64 of \0user\0pass>
Server: 235 Authentication successful

Client: MAIL FROM:<alice@a.com>
Server: 250 2.1.0 OK

Client: RCPT TO:<bob@b.com>
Server: 250 2.1.5 OK

Client: DATA
Server: 354 End data with <CR><LF>.<CR><LF>

Client: From: Alice <alice@a.com>
        To: Bob <bob@b.com>
        Subject: Hello
        
        Hi Bob!
        .

Server: 250 2.0.0 OK: queued as ABC123

Client: QUIT
Server: 221 2.0.0 Bye
```

---

## 5. 메시지 형식 (RFC 5322)

```
From: Alice <alice@a.com>
To: Bob <bob@b.com>
Cc: charlie@c.com
Subject: Hello
Date: Tue, 13 May 2026 12:00:00 +0900
Message-ID: <abc@a.com>
Reply-To: alice@a.com
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="X"

--X
Content-Type: text/plain; charset=UTF-8

Hi Bob!

--X
Content-Type: image/jpeg
Content-Transfer-Encoding: base64
Content-Disposition: attachment; filename="photo.jpg"

<base64 data>
--X--
```

### 표준 헤더
- **From / To / Cc / Bcc / Subject**
- **Date / Message-ID / Reply-To**
- **In-Reply-To / References** — 스레드
- **MIME-Version / Content-Type**

---

## 6. STARTTLS vs Implicit TLS

### STARTTLS (RFC 3207)
```
1. TCP 25 / 587 평문 연결
2. EHLO → 서버 STARTTLS 광고
3. STARTTLS 명령
4. TLS handshake
5. 이후 TLS 안 SMTP
```

#### 한계 — Downgrade attack
- MITM 가 STARTTLS 응답 변조 → 평문 강제
- 방어: MTA-STS, DANE

### Implicit TLS (465 / SMTPS)
- 시작부터 TLS — downgrade X
- 권장 (RFC 8314)

---

## 7. SMTP 응답 코드

| 코드 | 의미 |
| --- | --- |
| 2xx | 성공 |
| 220 | Service ready |
| 250 | OK |
| 251 | Forwarded |
| 3xx | 추가 정보 필요 |
| 354 | Start mail input |
| 4xx | 일시 실패 (retry) |
| 421 | Service unavailable |
| 450 | Mailbox busy |
| 452 | Insufficient storage |
| 5xx | 영구 실패 |
| 550 | Mailbox unavailable |
| 551 | User not local |
| 552 | Storage exceeded |
| 553 | Mailbox name invalid |
| 554 | Transaction failed |

---

## 8. 보안 / 인증

### AUTH 방식

| Mechanism | 의미 |
| --- | --- |
| PLAIN | Base64(user\0pass) — TLS 필수 |
| LOGIN | 두 단계 Base64 |
| CRAM-MD5 | Challenge-response (옛) |
| OAUTHBEARER | OAuth 2.0 — 모던 (Gmail / Outlook) |
| XOAUTH2 | OAuth 2.0 (옛 Gmail) |

### SPF / DKIM / DMARC
자세히 → [[spf-dkim-dmarc]]

### MTA-STS (RFC 8461)
- TLS 강제 — HTTPS 의 HSTS 와 같은 사상
- DNS TXT + HTTPS policy

### DANE (RFC 6698)
- DNS 의 TLSA record + DNSSEC
- TLS cert pinning

---

## 9. Bounce / DSN

### Bounce
- 메일 송신 실패 → 발신자에게 알림
- "Mailer-Daemon" 메일

### DSN — Delivery Status Notification (RFC 3464)
```
From: Mailer-Daemon@mail.example.com
Subject: Mail delivery failed
Content-Type: multipart/report; report-type=delivery-status

--X
Content-Type: message/delivery-status

Action: failed
Status: 5.1.1
Diagnostic-Code: smtp; 550 No such user
--X
```

---

## 10. Greylisting

```
첫 시도: 4xx (일시 실패)
재시도 (수 분 후): 250 (수락)
```

→ Spam bot 는 보통 재시도 X — spam 차단.

---

## 11. Postfix / Exim / Sendmail

### Postfix (모던 표준)
```
/etc/postfix/main.cf:
myhostname = mail.example.com
mydomain = example.com
smtpd_tls_cert_file = /etc/ssl/cert.pem
smtpd_tls_key_file = /etc/ssl/key.pem
smtpd_sasl_auth_enable = yes
smtp_use_tls = yes
```

### Sendmail
- 1983 — 가장 오래된 MTA
- 복잡한 설정 (.mc / m4)
- 사장 추세

### Exim
- Cambridge 의 MTA
- ACL 기반 강력
- Debian default (옛)

---

## 12. SMTP 의 라이브러리

### Python
```python
import smtplib
from email.message import EmailMessage

msg = EmailMessage()
msg['From'] = 'alice@a.com'
msg['To'] = 'bob@b.com'
msg['Subject'] = 'Hello'
msg.set_content('Hi Bob!')

with smtplib.SMTP('smtp.example.com', 587) as s:
    s.starttls()
    s.login('alice', 'password')
    s.send_message(msg)
```

### Node.js — nodemailer
```javascript
const nodemailer = require('nodemailer');
const transporter = nodemailer.createTransport({
    host: 'smtp.example.com',
    port: 587,
    secure: false,    // STARTTLS
    auth: {user: 'alice', pass: 'password'}
});

await transporter.sendMail({
    from: 'alice@a.com',
    to: 'bob@b.com',
    subject: 'Hello',
    text: 'Hi Bob!'
});
```

### Go
```go
import "net/smtp"
auth := smtp.PlainAuth("", "user", "pass", "smtp.example.com")
smtp.SendMail("smtp.example.com:587", auth, from, []string{to}, msg)
```

---

## 13. 디버깅 — telnet

```bash
telnet mail.example.com 25
220 mail.example.com ESMTP

EHLO myhost.com
250 ...

# 또는 openssl
openssl s_client -connect mail.example.com:465 -crlf
openssl s_client -connect mail.example.com:587 -crlf -starttls smtp
```

---

## 14. 함정

### 함정 1 — 25 port outbound 차단
ISP 가 spam 방지로 차단. 587 사용.

### 함정 2 — STARTTLS 의 downgrade
MITM 가능. MTA-STS / DANE.

### 함정 3 — Reverse DNS 없음
- 메일 서버 IP 의 PTR 없으면 차단
- ISP 에 설정

### 함정 4 — Open Relay
- 누구나 메일 송신 가능 → spam 도구화
- AUTH 필수

### 함정 5 — Spam score
- SPF / DKIM / DMARC 누락 → spam 폴더
- IP reputation 관리

### 함정 6 — Greylisting 의 retry
일부 응용 (트랜잭션) 의 retry 로직 — 즉시 재시도 X. 5-15 분 대기.

---

## 15. 학습 자료

- **RFC 5321** (SMTP)
- **RFC 5322** (Message format)
- **RFC 6409** (Submission)
- **RFC 8314** (Implicit TLS)
- **Postfix: The Definitive Guide** (Dent)

---

## 16. 관련

- [[mail]] — Mail hub
- [[pop3]] / [[imap]] — 수신
- [[spf-dkim-dmarc]] — 보안
- [[../dns/dns-record-types]] — MX / TXT
- [[../tls-ssl/tls-ssl]] — STARTTLS
