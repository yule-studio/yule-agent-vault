---
title: "SSH — 원격 접속 + 키 + 포워딩"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:00:00+09:00
tags:
  - operating-system
  - linux
  - ssh
---

# SSH

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | client / server / 키 / 포워딩 |

**[[linux|↑ Linux hub]]**

---

## 1. 한 줄

암호화된 원격 shell + 파일 전송 + 포트 포워딩. Linux 운영의 표준 입구.

---

## 2. Client

```bash
ssh user@host
ssh -p 2222 user@host
ssh -i ~/.ssh/id_ed25519 user@host
ssh -v user@host                       # verbose
ssh -A user@host                        # agent forwarding
ssh -X user@host                        # X11 forwarding
ssh user@host 'uptime'                  # 한 명령 실행
ssh user@host 'tail -f /var/log/app'    # 실시간

# 끝없는 세션
ssh -o ServerAliveInterval=60 user@host
```

---

## 3. 키 생성

```bash
ssh-keygen -t ed25519 -C "alice@example.com"
# 또는 RSA 4096
ssh-keygen -t rsa -b 4096
```

- `id_ed25519` (private) + `id_ed25519.pub` (public)
- passphrase 권장
- `~/.ssh/` 권한 700, 키 600

---

## 4. 키 등록

```bash
ssh-copy-id user@host
# 또는 수동
cat ~/.ssh/id_ed25519.pub | ssh user@host 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys'

ssh user@host    # 비밀번호 없이 로그인
```

`~/.ssh/authorized_keys` 권한: 600.

---

## 5. ssh-agent

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
ssh-add -l                              # 로딩된 키
ssh-add -d ~/.ssh/id_ed25519
```

passphrase 한 번만 입력. 데스크탑은 gnome-keyring / macOS Keychain 통합.

---

## 6. ~/.ssh/config

```ssh
Host prod
    HostName 10.0.0.5
    User alice
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
    ForwardAgent yes
    ServerAliveInterval 60

Host *.internal
    User admin
    ProxyJump bastion

Host bastion
    HostName bastion.example.com
    User alice

Host *
    AddKeysToAgent yes
    UseKeychain yes                     # macOS
    IdentitiesOnly yes
```

→ `ssh prod` 만으로 접속.

---

## 7. 서버 — sshd

`/etc/ssh/sshd_config`:

```
Port 22
PermitRootLogin no                      # ⚠️ no 권장
PasswordAuthentication no                # ✅ key only
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
AllowUsers alice bob
AllowGroups sudo
X11Forwarding no
AllowAgentForwarding no
MaxAuthTries 3
MaxSessions 5
ClientAliveInterval 300
ClientAliveCountMax 2
LoginGraceTime 30
Banner /etc/ssh/banner
LogLevel VERBOSE
KbdInteractiveAuthentication no
```

```bash
sudo sshd -t                            # config 검증
sudo systemctl restart sshd
```

---

## 8. 파일 전송

### 8.1 scp

```bash
scp file user@host:/path/
scp user@host:/path/file ./
scp -r dir user@host:/path/
scp -P 2222 ...
```

### 8.2 sftp

```bash
sftp user@host
> put file
> get file
> ls / cd / lcd ...
```

### 8.3 rsync (권장)

```bash
rsync -avzP src/ user@host:/path/        # 대부분의 경우
rsync -avzP --delete src/ host:/path/    # mirror
rsync -e 'ssh -p 2222' src/ host:/path/
```

resume / incremental / delete.

---

## 9. 포트 포워딩 (Tunnel)

### 9.1 Local Forward (-L)

```bash
ssh -L 8080:internal-host:80 user@bastion
# localhost:8080 → bastion → internal-host:80
```

원격 서비스를 로컬 포트로.

### 9.2 Remote Forward (-R)

```bash
ssh -R 9000:localhost:3000 user@host
# host:9000 → 내 localhost:3000
```

로컬 dev 서버 외부 노출 (ngrok 비슷).

### 9.3 Dynamic / SOCKS (-D)

```bash
ssh -D 1080 user@host
# localhost:1080 = SOCKS5 proxy → host 의 네트워크
```

브라우저 SOCKS 설정으로 회사 내부 망 접근.

### 9.4 ProxyJump (Bastion)

```bash
ssh -J bastion@gw:22 user@target
```

또는 config:
```
Host target
    ProxyJump bastion
```

---

## 10. ssh-agent forwarding

```bash
ssh -A user@host
```

또는 config `ForwardAgent yes`.

→ 원격에서 또 다른 ssh 시 로컬 키 사용. 편리하지만 **보안 위험** (compromised host 가 키 접근).

대안: ProxyJump 가 더 안전.

---

## 11. known_hosts / fingerprint

처음 접속 시:

```
The authenticity of host 'host (10.0.0.5)' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)?
```

→ `~/.ssh/known_hosts` 에 저장. 다음부터 자동.

서버 키 변경 시:
```bash
ssh-keygen -R hostname
```

---

## 12. Multiplexing

```ssh
# ~/.ssh/config
Host *
    ControlMaster auto
    ControlPath ~/.ssh/sockets/%r@%h:%p
    ControlPersist 10m
```

같은 host 의 첫 연결만 핸드셰이크, 이후는 같은 TCP 재사용 → 매우 빠른 ssh.

```bash
mkdir -p ~/.ssh/sockets
```

---

## 13. Mosh — 모바일 SSH

```bash
mosh user@host
```

- UDP, latency / 연결 끊김 강함
- 로컬 echo (typing latency ↓)
- 모바일 / 출장 친화

---

## 14. 보안 best practice

1. **PasswordAuthentication no** — key only
2. **PermitRootLogin no**
3. **AllowUsers / AllowGroups** 명시
4. **MaxAuthTries 3** + fail2ban
5. **port 변경** (22 → 2222) — security by obscurity 정도
6. ssh key passphrase + agent
7. ssh-key per server / per role (optional)
8. **2FA** (PAM Google Authenticator / YubiKey)
9. **CA-signed SSH key** (대규모 — Vault SSH)
10. ed25519 키 (RSA 보다 작고 빠름)

---

## 15. fail2ban

```bash
sudo apt install fail2ban

# /etc/fail2ban/jail.local
[sshd]
enabled = true
maxretry = 3
bantime = 1h
findtime = 10m
```

→ 실패 IP 자동 차단.

---

## 16. SSH CA / Certificates

```bash
# CA 키
ssh-keygen -f ssh_ca -t ed25519

# 사용자 키 서명
ssh-keygen -s ssh_ca -I "alice" -n alice,root -V +1d ~/.ssh/id_ed25519.pub
# → id_ed25519-cert.pub

# 서버 설정
TrustedUserCAKeys /etc/ssh/ssh_ca.pub
```

→ 한 CA 신뢰 = 모든 서명된 키 신뢰. enterprise / 짧은 expiration.

HashiCorp Vault, Smallstep, Teleport.

---

## 17. 자주 보는 문제

### 17.1 Permission denied (publickey)

```bash
ssh -v user@host
# 어떤 키를 보내고 어떤 인증 시도하는지

# 서버
sudo journalctl -u sshd
```

원인 — authorized_keys 권한, ~/.ssh 권한, key mismatch.

### 17.2 ssh hang
방화벽 / TCP keepalive. `ServerAliveInterval`.

### 17.3 Too many authentication failures
여러 키 로딩 → 서버 MaxAuthTries 초과:

```
ssh -o IdentitiesOnly=yes -i ~/.ssh/specific_key user@host
```

### 17.4 Host key verification failed
서버 키 변경. `ssh-keygen -R host` 후 재접속.

### 17.5 X forwarding 안 됨
`-X` 또는 `-Y`, `xauth` 설치, server 의 X11Forwarding yes.

---

## 18. 함정

### 18.1 password auth on internet
brute force 표적.

### 18.2 root key 보관
잃으면 lockout. 별도 admin 사용자 + sudo.

### 18.3 agent forwarding 의 위험
compromise host = 모든 키 노출. ProxyJump 권장.

### 18.4 ssh -A + 외부 host
민감 키 sharing.

### 18.5 known_hosts 의 hashing
`HashKnownHosts yes` — hostname 보호.

### 18.6 ssh into container
보통 잘못된 패턴. `docker exec`.

### 18.7 default key generation 옵션
`ssh-keygen` 만 = RSA 3072. **`-t ed25519`** 명시.

### 18.8 sshd_config 변경 후
**새 session 으로 검증** — 같은 session 유지 + `sshd -t`.

---

## 19. 학습 자료

- `man ssh`, `man sshd_config`, `man ssh_config`
- **OpenSSH** docs
- **SSH Mastery** — Michael W. Lucas

---

## 20. 관련

- [[../security/security]]
- [[networking]]
- [[linux]] — Linux hub
