---
title: "OS Security (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T15:30:00+09:00
tags:
  - operating-system
  - security
  - hub
---

# OS Security (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 4 개 세부 노트 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 한 줄

OS 의 격리 / 권한 / 접근 제어 메커니즘.
"누가 무엇을 할 수 있나" 를 강제.

---

## 2. 보안 layer

```
1. 사용자 / 권한 (UID/GID, mode bits)        ← 전통 UNIX
2. ACL                                         ← 더 세밀
3. Capability                                  ← root 의 권한 분리
4. SELinux / AppArmor (MAC)                    ← 의무적 접근 제어
5. seccomp                                     ← syscall 필터
6. Namespace / cgroup                          ← 격리
7. ASLR / NX / Stack canary / W^X              ← exploit 회피
8. KASLR / KPTI / Spectre mitigation           ← kernel 보호
```

---

## 3. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[permissions]] | UID/GID / mode / ACL / setuid |
| [[capabilities]] | Linux capability |
| [[selinux-apparmor]] | MAC (SELinux / AppArmor) |
| [[seccomp]] | syscall filter |
| [[sandbox]] | sandbox 패턴 |

---

## 4. UNIX 모델

```
파일마다: owner (uid), group (gid), other
권한: r/w/x × 3
```

자세히 → [[permissions]]

---

## 5. Capability

전통 UNIX: root (uid=0) = 모든 권한.
Linux capability: 41개 세분화된 권한 (CAP_NET_ADMIN, CAP_SYS_ADMIN, ...).

→ 응용에 정확히 필요한 권한만.

자세히 → [[capabilities]]

---

## 6. DAC vs MAC

| | DAC (Discretionary) | MAC (Mandatory) |
| --- | --- | --- |
| 의미 | 소유자가 권한 결정 | 시스템 정책 강제 |
| 예 | mode bits, ACL | SELinux, AppArmor |
| 우회 | sudo / 사회공학 | 정책 변경 권한 필요 |

DAC + MAC 조합이 강함.

자세히 → [[selinux-apparmor]]

---

## 7. seccomp — Syscall Filter

```c
// 허용 syscall 만 listed
// 나머지 → SIGKILL / SIGSYS / errno
```

응용 / container 의 attack surface 축소.

자세히 → [[seccomp]]

---

## 8. exploit 회피

| 기법 | 의미 |
| --- | --- |
| **ASLR** | 주소 랜덤화 → 공격자 주소 모름 |
| **NX / DEP** | 데이터 영역 실행 금지 |
| **W^X** | write 와 exec 동시 X |
| **Stack canary** | stack overflow 감지 |
| **PIE** | Position Independent Executable |
| **RELRO** | GOT read-only |
| **FORTIFY_SOURCE** | runtime 검사 |
| **CFI** | Control Flow Integrity |
| **KASLR** | kernel 도 |
| **KPTI** | Meltdown |
| **Retpoline / IBRS** | Spectre |
| **SMEP / SMAP** | kernel ↔ user 격리 |

`gcc -fstack-protector-strong -fPIE -pie -D_FORTIFY_SOURCE=2 -Wl,-z,now -Wl,-z,relro`.

---

## 9. 격리

| 메커니즘 | 의미 |
| --- | --- |
| **Namespace** | 자원 뷰 격리 |
| **cgroup** | 자원 제한 |
| **chroot / pivot_root** | rootfs 변경 |
| **VM** | hypervisor 격리 |
| **Seccomp** | syscall 제한 |

자세히 → [[sandbox]], [[../virtualization/namespace]]

---

## 10. PAM (Pluggable Authentication Modules)

```
/etc/pam.d/
  login, sshd, sudo, su, ...
```

인증 / 세션 / 패스워드 정책의 모듈식 구성.

```ini
auth     required  pam_unix.so
account  required  pam_unix.so
session  required  pam_limits.so
```

---

## 11. sudo / su

```bash
sudo -u user command          # 다른 사용자
sudo command                   # root
sudo -s                        # root shell
visudo                         # /etc/sudoers 편집
```

```
# /etc/sudoers
alice ALL=(ALL) NOPASSWD: /usr/bin/systemctl
```

→ root 권한 위임. PolicyKit 의 console 버전.

---

## 12. Audit

```bash
auditctl -w /etc/passwd -p wa -k passwd-change
ausearch -k passwd-change

# systemd
journalctl -u sshd
```

침해 감지 / 규정 준수.

---

## 13. SSH

```bash
ssh-keygen -t ed25519
ssh-copy-id user@host
# /etc/ssh/sshd_config
#   PasswordAuthentication no
#   PermitRootLogin no
```

자세히 → [[../linux/ssh]] (작성 예정)

---

## 14. 패스워드 / 인증

```
/etc/passwd           — 사용자 (UID, shell, home)
/etc/shadow           — bcrypt/sha512 hash 의 password
/etc/group / gshadow
```

PAM 으로 강도 정책. `chage` 로 만료.

---

## 15. 침해 대응 (Linux)

```bash
# 의심 process
ps auxf
ls -l /proc/*/exe
ss -tlnp

# 의심 파일
find / -mtime -1
find / -perm /4000              # setuid
chattr -i                        # immutable 풀기 (필요 시)

# 로그
journalctl
last
lastb
who
w

# 패키지 검증
debsums                          # debian
rpm -Va                          # redhat
```

---

## 16. 면접 핵심

1. **UID 0 vs capability**.
2. **setuid binary** — 위험과 사용.
3. **SELinux / AppArmor** — DAC vs MAC.
4. **seccomp** — 필요성.
5. **chroot vs namespace vs VM**.
6. **ASLR / NX / Canary**.
7. **Spectre / Meltdown / KPTI**.
8. **/etc/shadow** 의 해시.
9. **OOM Killer 가 위험한 시나리오**.
10. **container escape** 의 통로.

---

## 17. 학습 자료

- **Linux Security** — Bovet, Cesati Ch. 20
- **Container Security** — Liz Rice
- **The Linux Programming Interface** Ch. 38-39 (capability), 35 (process credentials)
- **Hardening Guide** — CIS, NSA, DISA

---

## 18. 관련

- [[../virtualization/namespace]]
- [[../virtualization/container]]
- [[../../security-theory/security-theory]]
- [[../operating-system|↑ OS hub]]
