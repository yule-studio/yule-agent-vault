---
title: "POP3 — Post Office Protocol v3"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T04:05:00+09:00
tags:
  - network
  - mail
  - pop3
---

# POP3 — Post Office Protocol v3

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RFC 1939 / 명령어 / IMAP 와 비교 |

**[[mail|↑ Mail]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **포트** | 110 (평문), 995 (POP3S) |
| **표준** | RFC 1939 (1996) |
| **모델** | Download-and-delete (옛) |
| **상태** | 사장 추세 (IMAP 대체) |

---

## 1. 한 줄 정의

**메일 수신** 의 옛 프로토콜. 메일을 클라이언트로 **다운로드 후 서버에서 삭제**.
모뎀 / 저장공간 부족하던 시대의 산물.

---

## 2. 동작 모델

```
1. 클라이언트 → 서버 (POP3) 접속
2. AUTH (USER / PASS)
3. LIST — 메일 목록
4. RETR — 메일 다운로드
5. DELE — 서버 메일 삭제
6. QUIT — 연결 종료
```

### 핵심
- **다운로드 후 서버 삭제** (옵션 — "Leave on server")
- 한 디바이스에 모든 메일 — 다른 디바이스 동기 X
- 폴더 / 라벨 개념 X (INBOX 만)

---

## 3. 명령

```
USER <name>        — 사용자 이름
PASS <password>    — 비밀번호
APOP <name> <md5>  — Challenge-response (옛)

STAT               — 메일 수 / 총 크기
LIST [n]           — 메일 목록 / 특정 메일 크기
RETR <n>           — 메일 가져오기
DELE <n>           — 메일 삭제 표시
NOOP               — 핑
RSET               — 삭제 표시 취소

TOP <n> <lines>    — 헤더 + 본문 일부
UID [n]            — 메일의 고유 ID
QUIT               — 종료 (DELE 적용)

STLS               — STARTTLS (RFC 2595)
```

---

## 4. 흐름

```
Client connect: TCP 110
Server: +OK POP3 ready

Client: USER alice
Server: +OK

Client: PASS secret
Server: +OK 3 messages

Client: STAT
Server: +OK 3 1500

Client: LIST
Server: +OK 3 messages
1 500
2 600
3 400
.

Client: RETR 1
Server: +OK 500 octets
<메일 본문>
.

Client: DELE 1
Server: +OK

Client: QUIT
Server: +OK Bye
```

---

## 5. POP3S vs STLS

### POP3S (995)
- Implicit TLS
- 시작부터 TLS

### STLS (110)
- STARTTLS — 평문 시작 → TLS 업그레이드
- Downgrade 공격 위험

### 권장
- POP3S (995) 사용 — RFC 8314

---

## 6. POP3 vs IMAP

| 항목 | POP3 | IMAP |
| --- | --- | --- |
| **저장** | 클라이언트 (다운로드) | 서버 |
| **다중 디바이스** | X (동기 X) | O |
| **폴더** | INBOX 만 | 다중 폴더 |
| **부분 다운로드** | X (전체) | O (헤더만 등) |
| **검색** | X | O (서버 검색) |
| **플래그** | X | Read/Flagged 등 |
| **오프라인** | 좋음 | 캐시 필요 |
| **포트** | 110 / 995 | 143 / 993 |

자세히 → [[imap]]

---

## 7. "Leave on server" 옵션

```
Client: RETR 1
Server: +OK <메일>
# DELE 안 함 → 서버에 남음
```

### 단점
- 서버 저장공간 부담
- 다중 디바이스 시 — 각 디바이스에 중복 다운로드 (동기 X)
- IMAP 가 본래 해결

---

## 8. APOP — 옛 인증

```
Server: +OK POP3 server ready <timestamp@host>
Client: APOP alice md5(<timestamp@host>secret)
Server: +OK
```

### 효과
- 평문 PASS 회피
- 단점 — MD5 약함, 평문 password 데이터베이스 필요

### 모던 — SASL
- AUTH PLAIN / LOGIN / OAUTHBEARER
- TLS 기반

---

## 9. POP3 의 라이브러리

### Python
```python
import poplib

server = poplib.POP3_SSL('pop.example.com', 995)
server.user('alice')
server.pass_('password')

num_messages = len(server.list()[1])
for i in range(num_messages):
    raw_message = b'\n'.join(server.retr(i+1)[1])
    print(raw_message.decode())

server.quit()
```

### Node.js — poplib
```javascript
const POP3Client = require('poplib');
const client = new POP3Client(995, 'pop.example.com', {tlserrs: false, enabletls: true});

client.on('connect', () => client.login('alice', 'password'));
client.on('login', (status) => {
    if (status) client.list();
});
```

---

## 10. 디버깅 — telnet / openssl

```bash
# 평문 110
telnet pop.example.com 110

# POP3S 995
openssl s_client -connect pop.example.com:995 -crlf

# STLS
openssl s_client -connect pop.example.com:110 -starttls pop3 -crlf

USER alice
PASS secret
LIST
RETR 1
QUIT
```

---

## 11. 함정

### 함정 1 — 평문 password
USER / PASS 가 평문 → TLS 필수.

### 함정 2 — DELE 후 RSET 불가
QUIT 이후 — DELE 영구 적용. RSET 은 QUIT 전에만.

### 함정 3 — 메일 수가 많을 때
LIST / RETR 가 느림. IMAP 의 부분 가져오기 권장.

### 함정 4 — 다중 디바이스 X
같은 메일 — 데스크탑 / 폰 각각 다운로드 → 동기 X.
모던 환경 — IMAP / Exchange 권장.

### 함정 5 — 옛 mailbox 잠금
POP3 — exclusive lock — 한 client 만 접속 가능. 동시 접속 시 fail.

---

## 12. 학습 자료

- **RFC 1939** (POP3)
- **RFC 2595** (STLS)
- **RFC 5034** (SASL)
- **RFC 8314** (Implicit TLS)

---

## 13. 관련

- [[mail]] — Mail hub
- [[imap]] — 모던 대체
- [[smtp]] — 송신
- [[../tls-ssl/tls-ssl]] — STARTTLS / STLS
