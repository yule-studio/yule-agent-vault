---
title: "Sandbox — 격리 패턴"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:55:00+09:00
tags:
  - operating-system
  - security
  - sandbox
---

# Sandbox

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | sandbox 기법 집계 |

**[[security|↑ Security hub]]**

---

## 1. 한 줄

신뢰할 수 없는 코드를 **제한된 환경에서 실행** — 격리 + 자원 제한 + 권한 박탈.

---

## 2. 격리 강도

```
강함  ─────────────────────────────  약함
 |              |              |              |
baremetal     VM           Light VM      Container     Process
                          (Firecracker)  (runc + seccomp)
```

---

## 3. Linux Sandbox 도구

| 도구 | 특징 |
| --- | --- |
| **chroot** | 옛 — rootfs 만 (보안 X) |
| **namespace + cgroup + seccomp + cap-drop** | 컨테이너 |
| **firejail** | desktop 응용 sandbox |
| **bubblewrap (bwrap)** | flatpak / xdg-desktop-portal |
| **systemd-nspawn** | systemd 의 container |
| **systemd 의 sandbox 옵션** | service 단위 |
| **Snap / Flatpak / AppImage** | 데스크탑 응용 |
| **gVisor** | 사용자 공간 kernel emul |
| **WASM runtime** | wasmtime / wasmer — 최강 격리 |

---

## 4. chroot — 옛 / 약함

```bash
chroot /sandbox /bin/bash
```

- rootfs 만 변경
- 같은 PID / network / IPC namespace
- **escape 쉬움** — 단순 보안 X

→ namespace 와 함께만 의미.

---

## 5. systemd-nspawn

```bash
sudo systemd-nspawn -D /var/lib/machines/mycontainer
sudo machinectl list
```

- chroot + 모든 namespace + cgroup
- container 의 systemd 친화
- 가벼운 VM 대체

---

## 6. systemd 의 Service Sandbox

`/etc/systemd/system/myapp.service`:

```ini
[Service]
ExecStart=/usr/bin/myapp

# 격리
User=myapp
Group=myapp
DynamicUser=yes
PrivateTmp=yes
PrivateNetwork=yes
PrivateDevices=yes
PrivateUsers=yes
ProtectSystem=strict
ProtectHome=true
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
ProtectClock=yes
ProtectHostname=yes
ProtectProc=invisible
ReadWritePaths=/var/log/myapp

# capability
CapabilityBoundingSet=
AmbientCapabilities=
NoNewPrivileges=yes

# seccomp
SystemCallFilter=@system-service
SystemCallErrorNumber=EPERM
SystemCallArchitectures=native

# 자원
MemoryMax=2G
CPUQuota=200%
TasksMax=512
```

→ container / VM 없이 **service 가 자체 sandbox**. systemd 가 강력.

---

## 7. Firejail

```bash
firejail firefox
firejail --net=none vim file.txt
```

- 데스크탑 응용
- profile 준비 (`/etc/firejail/`)
- 단순 사용

---

## 8. Bubblewrap

```bash
bwrap --ro-bind / / \
      --proc /proc --dev /dev --tmpfs /tmp \
      --unshare-all \
      /usr/bin/myapp
```

- 작고 단순
- Flatpak / GNOME 사용
- setuid 옵션 (rootless)

---

## 9. Container as Sandbox

container = 격리 + 자원 + 권한.

```bash
docker run --rm \
  --read-only \
  --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  --security-opt seccomp=profile.json \
  --network none \
  --user 1000 \
  ubuntu /bin/bash
```

자세히 → [[../virtualization/container]]

---

## 10. gVisor

```
응용의 syscall → gVisor sentry (user space) 에서 처리
→ host kernel 의 syscall 표면 ↓
→ 격리 ↑ (kernel exploit 어려움)
단, 성능 ↓
```

Google App Engine / GKE Sandbox 의 토대.

---

## 11. Kata Containers / Firecracker

각 컨테이너 = light VM:
- kata-runtime → containerd 의 OCI runtime
- container 와 같은 UX
- VM 의 격리 강도

→ multi-tenant SaaS.

자세히 → [[../virtualization/vm-hypervisor#7-light-vm]]

---

## 12. WASM Sandbox

```
WebAssembly = 격리 + cross-platform
wasmtime / wasmer / wamr 안에서 실행
WASI = file/network/clock 등 capability-based
```

가장 강한 user-mode 격리. K8s containerd-wasm-shim.

---

## 13. Browser Sandbox

웹 브라우저:
- multi-process 모델
- renderer = strict sandbox (seccomp, namespace, no FS, no network)
- broker process 가 mediator
- site isolation — 다른 origin = 다른 process

Chrome / Firefox / Safari 모두 비슷.

---

## 14. Mobile Sandbox

| | Sandbox |
| --- | --- |
| Android | per-app UID + SELinux + Binder + permission |
| iOS | per-app container + entitlement + signing |

→ desktop 보다 더 엄격.

---

## 15. 응용 / Library Sandbox

| | 특징 |
| --- | --- |
| **Java SecurityManager** | (deprecated 17+) policy 기반 |
| **Pyodide / WebAssembly** | Python in WASM |
| **WASI / WIT** | component model |
| **Rust isolate** | trait-based |
| **JS Worker** | per-worker context |

→ 응용 안에서도 plugin 격리.

---

## 16. Defense in Depth

한 layer 만 X — 여러 겹:

```
namespace → cgroup → capability drop → seccomp → AppArmor/SELinux
                                                  + read-only rootfs
                                                  + no-new-privileges
                                                  + (light VM)
```

→ 한 layer 깨져도 다른 layer 가 막음.

---

## 17. 함정

### 17.1 chroot 만 의존
escape 쉬움 (`fchdir` + ".." trick).

### 17.2 capability drop X
container 의 root 가 호스트 영향.

### 17.3 seccomp 없음
syscall 표면 그대로.

### 17.4 read-write rootfs
응용 손상 시 영구.

### 17.5 host network
격리 X.

### 17.6 secret in env
`ps` / docker inspect 에 노출.

### 17.7 sandbox 우회 보조 도구 미고려
ptrace, /proc, /sys, debugfs 등의 정보 노출.

### 17.8 multi-tenant + namespace 만
kernel exploit 위험 — light VM 권장.

---

## 18. 학습 자료

- **Container Security** — Liz Rice
- **The Chromium Project — Sandbox Design**
- **systemd.exec(5)**
- **OWASP Sandboxing Guidelines**
- **gVisor / Kata / Firecracker** docs

---

## 19. 관련

- [[capabilities]]
- [[seccomp]]
- [[selinux-apparmor]]
- [[../virtualization/namespace]]
- [[../virtualization/container]]
- [[security]] — Security hub
