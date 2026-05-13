---
title: "SSH — Secure Shell"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T05:05:00+09:00
tags:
  - network
  - ssh
  - security
---

# SSH — Secure Shell

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | RFC 4251 / 키 / 터널 / config |

**[[ssh-rdp-vpn|↑ 원격 접속]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **포트** | 22 / TCP |
| **표준** | RFC 4251 / 4252 / 4253 / 4254 (2006) |
| **계층** | L7 (TCP 위) |
| **암호화** | AES, ChaCha20-Poly1305 |
| **인증** | password / publickey / Kerberos |

---

## 1. 한 줄 정의

**안전한 원격 셸** — 1995 Tatu Ylönen. Telnet / rsh / rcp 의 암호화 대체. 명령 / 파일 / 터널.

---

## 2. SSH 의 3 레이어

```
SSH Transport (RFC 4253)    — 암호화 / 무결성 / 호스트 인증
        ↓
SSH Auth (RFC 4252)         — 사용자 인증
        ↓
SSH Connection (RFC 4254)   — 다중 채널 (셸 / 포트 / X11)
```

---

## 3. 핸드셰이크

```
Client → Server: TCP 22
Server: SSH-2.0-OpenSSH_8.9
Client: SSH-2.0-OpenSSH_9.0

# Key Exchange
Client / Server: KEXINIT (지원 알고리즘 광고)
DH key exchange (Curve25519 / ECDH)

# 호스트 인증
Server: host key (서명)
Client: known_hosts 와 비교

# 사용자 인증
Client: AUTH publickey/password
Server: 검증

# 채널 생성
Client: CHANNEL_OPEN session
Server: CHANNEL_OPEN_CONFIRMATION

# 셸 / 명령 실행
```

---

## 4. 키 인증

### 키 생성
```bash
ssh-keygen -t ed25519 -C "alice@example.com"
# 또는
ssh-keygen -t rsa -b 4096

# 결과
~/.ssh/id_ed25519       (private — 절대 공유 X)
~/.ssh/id_ed25519.pub   (public — 서버에 등록)
```

### 서버에 등록
```bash
# Method 1: ssh-copy-id
ssh-copy-id alice@server.com

# Method 2: 수동
cat ~/.ssh/id_ed25519.pub | ssh alice@server.com 'cat >> ~/.ssh/authorized_keys'
```

### 권장 알고리즘
- **Ed25519** — 짧고 빠르고 강함 (2014+)
- **RSA-4096** — 옛 호환 (느림)
- **ECDSA** — 가능 (NIST 곡선 비선호)

---

## 5. ssh_config / authorized_keys

### ~/.ssh/config (클라이언트)
```
Host work
    HostName server.example.com
    User alice
    Port 2222
    IdentityFile ~/.ssh/work_ed25519
    ForwardAgent yes

Host bastion-*
    ProxyJump bastion.example.com

Host internal-1.work
    ProxyJump bastion.example.com
```

→ `ssh work` — 단축.

### /etc/ssh/sshd_config (서버)
```
Port 22
PermitRootLogin no
PasswordAuthentication no          # 키만
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
X11Forwarding no
AllowUsers alice bob
```

### ~/.ssh/authorized_keys (서버 사용자)
```
ssh-ed25519 AAAA... alice@laptop
ssh-rsa AAAA... bob@server
```

### 옵션 — restriction
```
command="/usr/bin/rrsync /home/backup" ssh-ed25519 AAAA... backup-key
from="1.2.3.0/24" ssh-ed25519 AAAA... allowed-ip
no-port-forwarding,no-X11-forwarding ssh-ed25519 ...
```

---

## 6. SSH 의 채널 / 터널

### 6.1 Local Port Forward
```bash
ssh -L 8080:internal-db:5432 bastion
# 로컬 8080 → bastion 통해서 internal-db:5432
```

### 6.2 Remote Port Forward
```bash
ssh -R 9000:localhost:8080 jump-host
# jump-host 의 9000 → 내 로컬 8080
```

### 6.3 SOCKS Proxy
```bash
ssh -D 1080 bastion
# 로컬 1080 → bastion 의 SOCKS proxy
# 브라우저 — SOCKS5 localhost:1080
```

### 6.4 ProxyJump (모던)
```bash
ssh -J bastion internal-server
# 또는 config:
Host internal-server
    ProxyJump bastion
```

→ 옛 ProxyCommand 대체.

### 6.5 ProxyCommand (옛)
```bash
ssh -o ProxyCommand="ssh -W %h:%p bastion" internal-server
```

---

## 7. ssh-agent / agent forwarding

### ssh-agent
```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
```

→ 키 한 번만 unlock.

### Agent forwarding
```bash
ssh -A bastion
# bastion 에서 ssh internal-server → 로컬 agent 사용
```

### 보안
- **A 옵션 자제** — bastion 이 손상되면 키 사용 가능
- 대안 — **ProxyJump** (키 노출 X)

---

## 8. 파일 전송 — SCP / SFTP / rsync

### SCP (옛)
```bash
scp local.txt alice@server:/remote/
scp -r local-dir alice@server:/remote/
```

→ 사장 추세 (보안 약점 발견).

### SFTP (RFC draft)
```bash
sftp alice@server
sftp> put file.txt
sftp> get remote.txt
```

### rsync (권장)
```bash
rsync -av -e ssh local-dir/ alice@server:/remote/
rsync --delete -av local/ server:/remote/
```

→ 차이만 전송 — 빠름.

---

## 9. SSH 의 알고리즘

### Key Exchange
- curve25519-sha256
- ecdh-sha2-nistp256
- diffie-hellman-group14-sha256

### Cipher
- chacha20-poly1305@openssh.com
- aes256-gcm@openssh.com
- aes128-gcm@openssh.com

### MAC
- hmac-sha2-256-etm@openssh.com
- AEAD (GCM / ChaCha20-Poly1305) — MAC 불필요

### Public Key
- ssh-ed25519
- rsa-sha2-256 / rsa-sha2-512
- ecdsa-sha2-nistp256

### 확인
```bash
ssh -Q kex
ssh -Q cipher
ssh -Q mac
ssh -Q key
```

---

## 10. SSH 인증 — 방법

| 방법 | 설명 |
| --- | --- |
| **Password** | 옛 — brute force 위험 |
| **Publickey** | 권장 |
| **Keyboard-interactive** | 2FA (TOTP / U2F) |
| **GSSAPI/Kerberos** | 엔터프라이즈 |
| **Certificate** | CA 가 서명한 키 (모던) |

### SSH Certificate (모던)
```bash
# CA 가 사용자 키에 서명
ssh-keygen -s ca_key -I alice -n alice,admin -V +1d user_key.pub

# 결과
user_key-cert.pub
```

### 효과
- authorized_keys 관리 X (CA 가 서명한 cert 신뢰)
- 짧은 TTL (1 일)
- 회수 / revocation

### 사용 — Netflix BLESS, Smallstep, Teleport, HashiCorp Vault.

---

## 11. SSH 의 함정

### 함정 1 — Password auth 활성
brute force 표적. 키만 사용.

### 함정 2 — Root 로그인
PermitRootLogin no.

### 함정 3 — known_hosts 의 첫 접속
"Are you sure you want to continue?" — MITM 가능.
사전 host key 배포 (Ansible / cloud-init).

### 함정 4 — Agent forwarding 의 risk
bastion 손상 → 키 사용. ProxyJump 권장.

### 함정 5 — Port 22 brute force
fail2ban / port 변경 / 키 only.

### 함정 6 — TCP/IP 의 SSH 의 noisy
다양한 봇이 시도. log 가 큼.

### 함정 7 — Slowloris / 연결 폭주
MaxStartups / MaxSessions.

---

## 12. SSH Audit / 검사

### ssh-audit
```bash
ssh-audit example.com
# 알고리즘 / 키 강도 점검
```

### Lynis / openscap — 시스템 전반

---

## 13. 학습 자료

- **RFC 4251-4254** (SSH)
- **RFC 8709** (Ed25519)
- "SSH, the Secure Shell" (Barrett)
- OpenSSH man pages — `man ssh_config`, `man sshd_config`

---

## 14. 관련

- [[ssh-rdp-vpn]] — 원격 접속 hub
- [[../tls-ssl/tls-ssl]] — 비교 (TLS 도 PKI)
- [[../topics/zero-trust]] — Bastion 의 대체
