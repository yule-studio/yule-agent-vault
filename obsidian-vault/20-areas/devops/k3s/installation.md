---
title: "k3s installation — server / agent / systemd"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T09:02:00+09:00
tags: [devops, k3s, installation]
---

# k3s installation — server / agent / systemd

**[[k3s|↑ k3s]]**

---

## 1. 빠른 설치 (single node)

```bash
# 한 줄 — k3s server + agent 둘 다
curl -sfL https://get.k3s.io | sh -

# 확인
sudo systemctl status k3s
sudo k3s kubectl get nodes
```

→ 30초 안에 single-node cluster.

---

## 2. 정식 설치 (config 파일)

```bash
# 1. config 미리 작성
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml <<EOF
write-kubeconfig-mode: "0644"
tls-san:
  - k3s.example.com
disable:
  - traefik
  - servicelb
EOF

# 2. install (config 자동 인식)
curl -sfL https://get.k3s.io | sh -

# 3. 검증
sudo k3s kubectl cluster-info
```

→ command-line 옵션보다 **config 파일** 사용 권장 (재현 / drift 방지).

---

## 3. 특정 version 고정

```bash
# 표준 install script + INSTALL_K3S_VERSION
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.29.4+k3s1 sh -

# 또는 channel (stable / latest / v1.28)
curl -sfL https://get.k3s.io | INSTALL_K3S_CHANNEL=v1.29 sh -
```

→ production = 고정 version. `latest` X.

---

## 4. agent (worker) join

```bash
# server 에서 token 가져옴
SERVER=https://10.0.0.1:6443
TOKEN=$(ssh server "sudo cat /var/lib/rancher/k3s/server/node-token")

# agent install
curl -sfL https://get.k3s.io | K3S_URL=$SERVER K3S_TOKEN=$TOKEN sh -

# 확인 (server 에서)
sudo k3s kubectl get nodes
```

---

## 5. systemd service

```bash
sudo systemctl status k3s              # server
sudo systemctl status k3s-agent        # agent

# 로그
sudo journalctl -u k3s -f
sudo journalctl -u k3s-agent -f

# 재시작 (config 변경 후)
sudo systemctl restart k3s

# 부팅 자동 (default 켜져 있음)
sudo systemctl enable k3s
```

systemd unit 위치:
```
/etc/systemd/system/k3s.service
/etc/systemd/system/k3s.service.env       ← env (INSTALL_K3S_EXEC 인자)
```

→ 직접 편집 X. config.yaml 사용.

---

## 6. uninstall

```bash
# server
sudo /usr/local/bin/k3s-uninstall.sh

# agent
sudo /usr/local/bin/k3s-agent-uninstall.sh
```

→ data 까지 모두 제거. 재설치 깔끔.

---

## 7. k3sup (CLI 자동화 ★)

```bash
# 설치 (Go binary)
brew install k3sup

# server 설치 (SSH)
k3sup install \
    --ip 10.0.0.1 \
    --user ubuntu \
    --ssh-key ~/.ssh/id_ed25519 \
    --k3s-version v1.29.4+k3s1 \
    --tls-san k3s.example.com

# kubeconfig 자동 다운
ls kubeconfig

# agent join
k3sup join \
    --ip 10.0.0.2 \
    --server-ip 10.0.0.1 \
    --user ubuntu \
    --ssh-key ~/.ssh/id_ed25519
```

→ 다수 node 설치 자동화. CI / Terraform 친화.

---

## 8. air-gapped (offline) 설치

```bash
# online 환경에서:
# 1. binary 다운
curl -L https://github.com/k3s-io/k3s/releases/download/v1.29.4+k3s1/k3s -o k3s
curl -L https://get.k3s.io -o install.sh

# 2. image 다운 (containerd 가 미리 로드할)
curl -L https://github.com/k3s-io/k3s/releases/download/v1.29.4+k3s1/k3s-airgap-images-amd64.tar.zst -o images.tar.zst

# 3. 옮김 (USB / 내부망)

# 4. air-gapped node 에서:
sudo mkdir -p /var/lib/rancher/k3s/agent/images
sudo mv images.tar.zst /var/lib/rancher/k3s/agent/images/
sudo mv k3s /usr/local/bin/
sudo chmod +x /usr/local/bin/k3s

# 5. install (offline)
INSTALL_K3S_SKIP_DOWNLOAD=true sh install.sh
```

→ 선박 / 군 / 격리망.

---

## 9. private registry 통합

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  docker.io:
    endpoint:
      - "https://registry.internal.example.com/v2/docker"

  registry.internal.example.com:
    endpoint:
      - "https://registry.internal.example.com"

configs:
  registry.internal.example.com:
    auth:
      username: deploy
      password: secret
    tls:
      cert_file: /etc/ssl/registry.crt
      key_file: /etc/ssl/registry.key
      ca_file: /etc/ssl/ca.crt
```

→ Docker Hub rate limit 회피 / 회사 image.

---

## 10. resource 요구 (★ 시니어)

| Node 종류 | CPU | RAM | Disk |
| --- | --- | --- | --- |
| **server (single)** | 1 vCPU | 512MB | 4GB |
| **server (HA, 100 pod)** | 2 vCPU | 2GB | 16GB |
| **server (HA, 1000 pod)** | 4 vCPU | 8GB | 32GB |
| **agent (10 pod)** | 1 vCPU | 1GB | 8GB |
| **agent (50 pod)** | 4 vCPU | 8GB | 32GB |

→ ARM (RPi 4 4GB) 도 OK.

---

## 11. ARM / Apple Silicon

```bash
# RPi (ARM64) — automatic detection
curl -sfL https://get.k3s.io | sh -

# 32-bit ARM (older RPi) — boot/cmdline.txt 에 cgroup 활성
cgroup_memory=1 cgroup_enable=memory
```

→ k3s 의 official 지원 (ARM64 + AMD64).

---

## 12. 함정

1. **`latest` 사용** — breaking change 위험.
2. **token 노출** — `/var/lib/rancher/k3s/server/token` 권한 600.
3. **time sync 안 맞음** — NTP 필수 (TLS / etcd 영향).
4. **kubeconfig 권한 644** — 작은 환경 OK, 보안 환경 600.
5. **single server + production** — HA 없으면 단일 장애.
6. **install script 직접 다운로드** — 일부 환경 mirror 필요.
7. **firewall 미설정** — 6443 (apiserver), 10250 (kubelet), 8472 UDP (VXLAN), 51820 UDP (WireGuard) 열어야.

---

## 13. 관련

- [[k3s|↑ k3s]]
- [[concepts]]
- [[ha-mode]]
- [[networking]]
