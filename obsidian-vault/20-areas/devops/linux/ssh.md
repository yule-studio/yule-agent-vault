---
title: "SSH — key / config / agent / tunnel"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:51:00+09:00
tags: [devops, linux, ssh]
---

# SSH — key / config / agent / tunnel

**[[linux|↑ linux]]**

---

## 1. 기본

```bash
ssh user@host                     # 22 port
ssh -p 2222 user@host
ssh -i ~/.ssh/work_id user@host   # 특정 key
ssh -L 8080:localhost:80 user@host    # local forward
ssh -R 8080:localhost:80 user@host    # remote forward
ssh -D 1080 user@host                  # SOCKS proxy
```

---

## 2. key 생성 / 복사

```bash
# ed25519 권장 (RSA 보다 빠르고 작음)
ssh-keygen -t ed25519 -C "alice@laptop"

# RSA 도 OK (4096)
ssh-keygen -t rsa -b 4096

# 결과
~/.ssh/id_ed25519        # private (600)
~/.ssh/id_ed25519.pub    # public (644)

# 서버에 등록
ssh-copy-id user@host
# 또는 manual:
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

---

## 3. ~/.ssh/config

```
# ~/.ssh/config
Host *
    ServerAliveInterval 60
    AddKeysToAgent yes
    UseKeychain yes              # macOS

Host prod-bastion
    HostName bastion.example.com
    User alice
    Port 2222
    IdentityFile ~/.ssh/prod_id_ed25519

Host prod-*
    User app
    ProxyJump prod-bastion       # bastion 통해서
    IdentityFile ~/.ssh/prod_id_ed25519

Host work-github
    HostName github.com
    User git
    IdentityFile ~/.ssh/work_id_ed25519
```

→ `ssh prod-web1` 만 쳐도 OK.

---

## 4. ssh-agent

```bash
eval $(ssh-agent)
ssh-add ~/.ssh/id_ed25519
ssh-add -l                       # 등록된 key 목록
ssh-add -D                       # 전부 제거

# macOS
ssh-add --apple-use-keychain ~/.ssh/id_ed25519

# agent forwarding (조심)
ssh -A user@bastion              # 본인 agent 를 bastion 에서 사용
# config:
ForwardAgent yes
```

→ **ForwardAgent 는 보안 위험** (해당 서버 root 가 본인 key 탈취 가능). bastion 신뢰 시만.

---

## 5. server 설정 (/etc/ssh/sshd_config)

```
# 권장 설정
Port 22                          # 또는 2222
PermitRootLogin no               # ★
PasswordAuthentication no        # ★ key only
PubkeyAuthentication yes
ChallengeResponseAuthentication no
UsePAM yes
AllowUsers alice bob             # 또는 AllowGroups admins
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
ClientAliveInterval 300
ClientAliveCountMax 2

# 강한 algorithm 만
KexAlgorithms curve25519-sha256@libssh.org,curve25519-sha256
HostKeyAlgorithms ssh-ed25519,rsa-sha2-512
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com

# X11 / TCP forwarding (필요 시 키기)
X11Forwarding no
AllowTcpForwarding no
GatewayPorts no
```

```bash
sudo sshd -t                      # config 검증
sudo systemctl reload ssh         # 적용
```

---

## 6. tunneling

### 6-1. local port forward

```bash
# local:8080 → remote:80 (host 가 SSH 서버)
ssh -L 8080:localhost:80 user@host

# 또는 bastion 통해 내부 DB
ssh -L 5432:db.internal:5432 user@bastion
# → localhost:5432 ↔ db.internal:5432 (bastion 경유)
```

### 6-2. remote port forward

```bash
# remote:9090 → local:3000 (개발 PC 를 외부 노출)
ssh -R 9090:localhost:3000 user@host
```

### 6-3. SOCKS proxy

```bash
ssh -D 1080 user@host
# → localhost:1080 이 SOCKS5
# 브라우저 / curl: --socks5 localhost:1080
curl --socks5 localhost:1080 https://internal.example.com
```

---

## 7. SCP / SFTP / rsync

```bash
# scp (간단)
scp file user@host:/path
scp -r dir user@host:/path
scp user@host:/path/file ./

# rsync (★ 권장 — 차분 / resume)
rsync -avz file user@host:/path
rsync -avz --delete dir/ user@host:/path/    # mirror
rsync --progress -avzh src/ user@host:/dst/

# sftp (interactive)
sftp user@host
> cd /var/log
> get *.log
```

---

## 8. troubleshoot

```bash
ssh -vvv user@host                # verbose (디버그)

# 흔한 에러
# Permission denied (publickey)
#   → key 있나? ssh-add -l / config IdentityFile
#   → 서버의 authorized_keys 권한 600?
#   → ~/.ssh 권한 700?

# Connection refused
#   → 서버 sshd 동작? systemctl status ssh
#   → 방화벽? firewall-cmd / ufw / security group

# Host key verification failed
#   → ~/.ssh/known_hosts 의 fingerprint 변경
#   → ssh-keygen -R host  (제거 후 재확인)

# Too many authentication failures
#   → ssh-agent 가 여러 key 시도 → IdentitiesOnly yes
```

---

## 9. bastion / jump host

```
[laptop] ─ssh─→ [bastion] ─ssh─→ [private servers]
                  공인 IP            10.0.x.x

# 방법 1: ProxyJump (권장)
ssh -J alice@bastion app@private-server

# 방법 2: config
Host prod-*
    ProxyJump bastion

# 방법 3: ProxyCommand (legacy)
ProxyCommand ssh -W %h:%p bastion
```

---

## 10. AWS / cloud SSH

```bash
# AWS Systems Manager Session Manager (SSH 없이)
aws ssm start-session --target i-1234567890abcdef0

# GCP IAP (Identity-Aware Proxy)
gcloud compute ssh instance-name --tunnel-through-iap

# EC2 Instance Connect
aws ec2-instance-connect send-ssh-public-key ...
```

→ bastion 불필요, 감사 log 자동.

---

## 11. 보안 best practice

```
1. PasswordAuthentication no
2. PermitRootLogin no
3. key 만 사용 (ed25519)
4. private key 에 passphrase
5. ssh-agent 사용 (passphrase 한 번만)
6. ForwardAgent 신뢰하는 host 만
7. fail2ban 설치 (brute-force 차단)
8. 가능하면 SSM / IAP 사용 (key 관리 불필요)
9. 정기 key rotation
10. audit log (auth.log) 모니터링
```

---

## 12. 함정

1. **password auth 켜져있음** — brute-force 표적.
2. **`chmod 644 ~/.ssh/id_rsa`** — ssh client 거부.
3. **ForwardAgent 항상 켬** — agent hijack 위험.
4. **bastion 없이 모든 server 가 공인 IP** — 공격면 ↑.
5. **공유 key** — 누가 접근했는지 모름. 사용자별 key.
6. **`~/.ssh/known_hosts` 무시** — MITM 공격 시 알아챔 X.

---

## 13. 관련

- [[linux|↑ linux]]
- [[file-permissions]]
- [[networking-basics]]
- [[../security-ops/security-ops|↗ security-ops]]
