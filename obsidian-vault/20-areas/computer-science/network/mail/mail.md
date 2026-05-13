---
title: "Email Protocols — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T03:55:00+09:00
tags:
  - network
  - mail
  - smtp
  - imap
---

# Email Protocols — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | SMTP / POP3 / IMAP / SPF/DKIM/DMARC |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/layer-7-application/layer-7-application|↑↑ L7]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **포트** | 25 (SMTP), 465 (SMTPS), 587 (Submission), 110 (POP3), 995 (POP3S), 143 (IMAP), 993 (IMAPS) |
| **표준** | RFC 5321 (SMTP), 9051 (IMAP), 1939 (POP3) |
| **시대** | SMTP 1982 (RFC 821) |

---

## 1. 4 가지 프로토콜

| 프로토콜 | 용도 | 포트 |
| --- | --- | --- |
| **SMTP** | 메일 송신 (서버 간 + 클라→서버) | 25 / 465 / 587 |
| **POP3** | 메일 수신 (다운로드) | 110 / 995 |
| **IMAP** | 메일 수신 (서버 동기) | 143 / 993 |
| **MAPI / EWS** | Microsoft (Exchange) | — |

---

## 2. 흐름

```
Alice (alice@a.com) 가 Bob (bob@b.com) 에게 메일

1. Alice 의 메일 클라이언트 → Alice 의 SMTP server (587)
   AUTH + 메일 송신
2. Alice 의 SMTP → Bob 의 SMTP (25)
   MX record 조회 → mail.b.com 으로
3. Bob 의 SMTP → mailbox 에 저장
4. Bob 의 클라이언트 ← Bob 의 IMAP / POP3 (993)
   조회 / 다운로드
```

### Mail User Agent (MUA)
- 클라이언트 (Outlook, Mail.app, Thunderbird, Gmail web)

### Mail Submission Agent (MSA)
- 587 port — 클라이언트 → 서버

### Mail Transfer Agent (MTA)
- 25 port — 서버 간 (Postfix, Sendmail, Exim)

### Mail Delivery Agent (MDA)
- 사용자 mailbox 에 저장 (Dovecot)

---

## 3. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[smtp]] | SMTP — 송신 |
| [[pop3]] | POP3 — 옛 수신 |
| [[imap]] | IMAP — 모던 수신 |
| [[spf-dkim-dmarc]] | 메일 인증 / 위조 방지 |

---

## 4. 보안

| 위협 | 방어 |
| --- | --- |
| **Spam** | SPF / DKIM / DMARC / Bayesian filter |
| **Spoofing** | SPF / DKIM / DMARC |
| **Phishing** | DMARC / 사용자 교육 |
| **Eavesdropping** | STARTTLS / SMTPS / IMAPS |
| **Account takeover** | MFA / 강한 비밀번호 |
| **Brute force** | Rate limit / Fail2Ban |

자세히 → [[spf-dkim-dmarc]]

---

## 5. 메일 서비스 (SaaS)

### 비즈니스
- **Google Workspace** (Gmail)
- **Microsoft 365** (Exchange Online)
- **Zoho Mail**
- **ProtonMail** — 암호화
- **FastMail** — IMAP / 표준 친화

### 전송 (Transactional)
- **SendGrid** (Twilio)
- **Mailgun**
- **Amazon SES**
- **Postmark**

### 마케팅
- **Mailchimp**
- **Klaviyo**

---

## 6. 자체 운영 — 거의 X

### 어려움
- Spam reputation — IP 가 spammer 리스트 등록되면 차단
- 운영 / 보안 / 백업
- SPF / DKIM / DMARC / Reverse DNS / blacklisted IP 확인

### 옛 SMTP
- Sendmail (1983) — 설정 복잡
- Postfix — 모던 / 안정
- Exim — 유연

### 모던
- 99% 가 SaaS 사용
- 자체 운영 — 큰 회사 / 정부

---

## 7. 면접 / 토픽

1. **SMTP / POP3 / IMAP** 차이.
2. **SPF / DKIM / DMARC**.
3. **메일이 spam 으로 분류되는 이유**.
4. **MX record 의 동작**.
5. **STARTTLS vs SMTPS**.

---

## 8. 학습 자료

- **RFC 5321** (SMTP)
- **RFC 9051** (IMAP)
- **RFC 1939** (POP3)
- **RFC 7208** (SPF), **RFC 6376** (DKIM), **RFC 7489** (DMARC)
- "Postfix: The Definitive Guide"
- mail-tester.com — 메일 spam score

---

## 9. 관련

- [[../osi-7-layer/layer-7-application/layer-7-application]] — L7
- [[../dns/dns-record-types]] — MX / TXT (SPF/DKIM/DMARC)
- [[../tls-ssl/tls-ssl]] — STARTTLS
