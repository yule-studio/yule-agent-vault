---
title: "IMAP — Internet Message Access Protocol"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T04:10:00+09:00
tags:
  - network
  - mail
  - imap
---

# IMAP — Internet Message Access Protocol

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RFC 9051 / 명령어 / IDLE / 폴더 |

**[[mail|↑ Mail]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **포트** | 143 (평문), 993 (IMAPS) |
| **표준** | RFC 9051 (2021) — IMAP4rev2 |
| **모델** | Server-side (동기) |
| **사용** | 모던 메일 클라이언트 |

---

## 1. 한 줄 정의

**서버 중심** 의 메일 수신 프로토콜. 메일을 서버에 두고 클라이언트가 **부분 조회 / 폴더 관리 / 검색** 가능. 다중 디바이스 동기.

---

## 2. 동작 모델

```
Server <─> Client (Mail.app)
       <─> Client (iPhone)
       <─> Client (Gmail web)

각 클라이언트가 같은 서버에 접속 → 같은 상태 (읽음 / 폴더 / 라벨 동기)
```

### 핵심
- **서버에 저장** (다운로드 X)
- **다중 폴더** (INBOX, Sent, Drafts, Trash, ...)
- **플래그** (\\Seen, \\Flagged, \\Answered, \\Deleted)
- **부분 다운로드** — 헤더만 / 본문 일부 / 첨부 별도
- **검색 / 정렬 / sort** — 서버 측 수행

---

## 3. IMAP 의 상태

```
1. Not Authenticated
2. Authenticated     ← LOGIN / AUTH
3. Selected          ← SELECT INBOX
4. Logout            ← LOGOUT
```

---

## 4. 명령

### Authentication
```
LOGIN <user> <pass>
AUTHENTICATE PLAIN / LOGIN / XOAUTH2 / OAUTHBEARER
STARTTLS
```

### Mailbox
```
SELECT INBOX            — 열기
EXAMINE INBOX           — 읽기 전용
LIST "" "*"             — 폴더 목록
CREATE Archive
DELETE Old
RENAME Old New
SUBSCRIBE / UNSUBSCRIBE
```

### Message
```
FETCH 1:10 (FLAGS BODY[HEADER])      — 헤더만
FETCH 1 BODY[]                       — 전체
FETCH 1 BODY[1]                      — 첫 part
STORE 1 +FLAGS \\Seen                — 플래그 설정
COPY 1:5 Archive
MOVE 1:5 Archive
EXPUNGE                              — \\Deleted 영구 삭제
```

### Search
```
SEARCH FROM "alice@a.com"
SEARCH UNSEEN
SEARCH SINCE 1-Jan-2026
SEARCH SUBJECT "Hello"
SEARCH BODY "keyword"
```

### IDLE — Push
```
IDLE                    — 서버가 변경 시 알림 (RFC 2177)
                          ← * 1 EXISTS (새 메일)
DONE                    — IDLE 종료
```

---

## 5. 흐름

```
Client connect: TCP 993 (TLS)
Server: * OK [CAPABILITY IMAP4rev2 STARTTLS AUTH=PLAIN] Ready

Client: a001 LOGIN alice password
Server: a001 OK LOGIN completed

Client: a002 SELECT INBOX
Server: * 50 EXISTS
        * 5 RECENT
        * OK [UIDVALIDITY 1234567890]
        * OK [UIDNEXT 51]
        * FLAGS (\\Answered \\Flagged \\Deleted \\Seen)
        a002 OK [READ-WRITE] SELECT completed

Client: a003 FETCH 1:5 (FLAGS ENVELOPE)
Server: * 1 FETCH (FLAGS (\\Seen) ENVELOPE (...))
        * 2 FETCH ...
        a003 OK FETCH completed

Client: a004 FETCH 1 BODY[]
Server: * 1 FETCH (BODY[] {1234}
        <전체 메일>)
        a004 OK FETCH completed

Client: a005 IDLE
Server: + idling
        ... (대기)
        * 51 EXISTS    ← 새 메일 도착

Client: DONE
Server: a005 OK IDLE terminated

Client: a006 LOGOUT
Server: * BYE
        a006 OK LOGOUT completed
```

---

## 6. UID — Unique Identifier

```
* 1 EXISTS    ← Sequence number (1 부터, 변동)
UIDNEXT 51    ← UID 51 부터 (영구)
```

### UID vs Sequence
- **Sequence** — 1, 2, 3, ... (메일 삭제 시 변동)
- **UID** — 영구 (메일 별 고유)

### UID FETCH
```
UID FETCH 100:200 (FLAGS)
```

→ 클라이언트가 캐시하기 좋게.

---

## 7. 폴더 / 라벨

### 폴더 (Folder)
- INBOX, Sent, Drafts, Trash, Archive
- 한 메일이 한 폴더에만

### 라벨 (Gmail)
- IMAP 표준 외 — Gmail 의 라벨 확장
- 한 메일이 여러 라벨

### 표준 명
- INBOX (대문자, 표준)
- Sent / Sent Items / Sent Mail (서버마다 다름)
- 클라이언트 — 자동 매핑

---

## 8. IMAP IDLE — Push notification

```
Client: a001 IDLE
Server: + idling

(서버가 새 메일 / 변경 발생 시)
Server: * 51 EXISTS
Server: * 5 RECENT

Client: DONE
Server: a001 OK IDLE terminated
```

### 효과
- 폴링 없이 실시간 알림
- 모바일 — 배터리 절약

### 한계
- TCP 연결 유지 — 일부 라우터 / NAT 가 끊음
- Mobile push 는 APNs / FCM 가 더 효율 (Gmail 의 X-Mailer)

---

## 9. IMAPS vs STARTTLS

### IMAPS (993)
- Implicit TLS

### STARTTLS (143)
- RFC 2595
- Downgrade 공격 위험

### 권장
- IMAPS (993) — RFC 8314

---

## 10. IMAP 의 라이브러리

### Python — imaplib
```python
import imaplib

mail = imaplib.IMAP4_SSL('imap.example.com', 993)
mail.login('alice', 'password')
mail.select('INBOX')

# 검색
typ, data = mail.search(None, 'UNSEEN')
for num in data[0].split():
    typ, msg = mail.fetch(num, '(RFC822)')
    print(msg[0][1].decode())

mail.logout()
```

### Node.js — imapflow / node-imap
```javascript
import {ImapFlow} from 'imapflow';

const client = new ImapFlow({
    host: 'imap.example.com',
    port: 993,
    secure: true,
    auth: {user: 'alice', pass: 'password'}
});

await client.connect();
await client.mailboxOpen('INBOX');

for await (const message of client.fetch('1:*', {envelope: true})) {
    console.log(message.envelope);
}

await client.logout();
```

### Go — emersion/go-imap
```go
import "github.com/emersion/go-imap/v2/imapclient"

c, _ := imapclient.DialTLS("imap.example.com:993", nil)
c.Login("alice", "password").Wait()
c.Select("INBOX", nil).Wait()
```

---

## 11. JMAP — IMAP 의 모던 대체

### JMAP (RFC 8620, 2019)
- HTTP / JSON 기반
- FastMail 가 설계 / 운영
- IMAP 의 복잡성 / 비효율 개선

### 흐름
```
POST /jmap HTTP/1.1
Authorization: Bearer ...

{
  "using": ["urn:ietf:params:jmap:mail"],
  "methodCalls": [
    ["Email/query", {"filter": {"inMailbox": "abc"}}, "1"]
  ]
}
```

### 효과
- 배치 (한 요청에 여러 작업)
- WebSocket / push
- 모바일 / web 친화

### 도입
- FastMail (모든 사용자)
- Topicbox / 일부 메일 서비스
- 메이저 — 아직 IMAP

---

## 12. 디버깅 — openssl

```bash
openssl s_client -connect imap.example.com:993 -crlf

a001 LOGIN alice password
a002 SELECT INBOX
a003 FETCH 1 BODY[]
a004 LOGOUT
```

---

## 13. 함정

### 함정 1 — UIDVALIDITY 변경
서버가 INBOX 의 UIDVALIDITY 변경 → 클라이언트 캐시 무효.
재동기 필요.

### 함정 2 — IDLE 연결 끊김
NAT / 방화벽 가 30 분 후 끊음.
주기적 NOOP (10-25 분 마다).

### 함정 3 — Gmail 의 라벨 ↔ 폴더 매핑
"All Mail" 가 모든 메일 — 한 메일이 여러 폴더에 보일 수 있음.

### 함정 4 — Sent 메일의 중복
SMTP send + IMAP APPEND to Sent — Gmail / Outlook 가 자동 처리.
다른 서버 — 클라이언트가 명시 APPEND.

### 함정 5 — 검색 성능
서버 측 — 백만 메일 검색 느림.
ElasticSearch / Solr 기반 검색 인덱스 필요 (Dovecot FTS plugin).

### 함정 6 — Push 의 비표준
IMAP IDLE 만 표준. Gmail / Apple — APNs / FCM 의 별도 push.

---

## 14. 학습 자료

- **RFC 9051** (IMAP4rev2, 2021)
- **RFC 3501** (IMAP4rev1, 옛)
- **RFC 2177** (IDLE)
- **RFC 8620** (JMAP)
- "Dovecot Documentation"

---

## 15. 관련

- [[mail]] — Mail hub
- [[pop3]] — 옛 대안
- [[smtp]] — 송신
- [[../tls-ssl/tls-ssl]] — STARTTLS
