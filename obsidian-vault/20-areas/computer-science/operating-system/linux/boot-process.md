---
title: "Linux 부팅 과정 — BIOS/UEFI → bootloader → kernel → systemd"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:15:00+09:00
tags:
  - operating-system
  - linux
  - boot
---

# Linux 부팅 과정

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | UEFI / GRUB / initramfs / systemd |

**[[linux|↑ Linux hub]]**

---

## 1. 흐름

```
1. 전원 ON
2. UEFI/BIOS POST
3. UEFI firmware → ESP 의 bootloader 실행
   (또는 BIOS → MBR 의 GRUB)
4. GRUB (bootloader) → kernel + initramfs 로드
5. Kernel 초기화 — root fs 마운트 (initramfs 도움)
6. /sbin/init 실행 — 보통 systemd
7. systemd → target 도달 → service 시작
8. login prompt / GUI
```

---

## 2. UEFI vs BIOS

| | BIOS (legacy) | UEFI |
| --- | --- | --- |
| 디스크 | MBR (max 2 TB) | GPT (max 8 ZB) |
| Boot | MBR 의 446 byte | ESP 의 .efi 파일 |
| Secure Boot | 없음 | 서명된 binary 만 |
| 다중 OS | bootloader 가 관리 | UEFI menu |
| 현재 | 옛 시스템만 | 표준 |

```bash
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "BIOS"
```

---

## 3. ESP — EFI System Partition

```bash
ls /boot/efi/EFI
# BOOT/                  fallback
# ubuntu/, fedora/, debian/, ...
# Microsoft/             dual-boot

ls /boot/efi/EFI/ubuntu
# grubx64.efi
# shimx64.efi          (Secure Boot)
# mmx64.efi
```

UEFI firmware 가 `EFI/<vendor>/grubx64.efi` 실행.

---

## 4. GRUB (GRand Unified Bootloader)

```bash
# 설정
/etc/default/grub
sudo update-grub                # Debian/Ubuntu
sudo grub2-mkconfig -o /boot/grub2/grub.cfg     # RHEL/Fedora

# /etc/default/grub
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_CMDLINE_LINUX="quiet splash"
GRUB_CMDLINE_LINUX_DEFAULT=""
```

GRUB 시작 시 menu / kernel 선택 → kernel + initrd 로드.

---

## 5. Kernel cmdline

```bash
cat /proc/cmdline
# BOOT_IMAGE=/vmlinuz-6.5.0 root=UUID=... ro quiet splash
```

자주 쓰는 옵션:
- `root=UUID=...` — root fs
- `ro` / `rw` — initial mount
- `quiet` / `verbose`
- `nosplash`
- `single` / `emergency`
- `init=/bin/bash` — 디버그 (init 교체)
- `nomodeset` — graphics 문제
- `mitigations=off` — Spectre 등 (비추)
- `panic=10` — panic 시 reboot
- `isolcpus=2-3` — CPU 격리

---

## 6. initramfs / initrd

```
초기 root fs — RAM 위. driver, fs module, decryption 도구 포함.
진짜 root fs 준비 → pivot_root.
```

```bash
ls /boot/initrd.img-*
lsinitramfs /boot/initrd.img-$(uname -r) | head
# init / lib / etc / bin / sbin / dev / proc / sys

# 재생성
sudo update-initramfs -u             # Debian/Ubuntu
sudo dracut -f                        # RHEL/Fedora
```

이 단계의 역할:
- 디스크 driver 로드
- LUKS / LVM / RAID 해결
- 진짜 root mount

---

## 7. Kernel 초기화

```
1. BIOS/UEFI 가 kernel image 메모리에 로드
2. kernel 압축 해제
3. early boot (arch-specific)
4. start_kernel():
   - schedule init
   - mm init
   - VFS init
   - 모든 subsystem init
5. driver / module 로드 (initramfs 의)
6. rootfs 마운트
7. /sbin/init (PID 1) 실행
```

`dmesg | head -50` 으로 부팅 첫 단계 메시지.

---

## 8. /sbin/init = systemd (대부분)

```bash
ls -l /sbin/init
# /sbin/init -> /lib/systemd/systemd
```

PID 1. 자세히 → [[systemd]]

---

## 9. systemd 의 target

```
default.target → multi-user.target → ...

graphical.target → multi-user.target → basic.target → sysinit.target ...
```

- `default.target` — 부팅 후 목표
- 의존성 그래프 + 병렬 시작
- service 들이 sysinit.target 후 시작

```bash
systemctl get-default          # graphical.target / multi-user.target
systemctl set-default multi-user.target

systemctl list-units --type=target
```

---

## 10. boot 분석

```bash
systemd-analyze
# Startup finished in 1.234s (kernel) + 5.678s (userspace) = 6.912s

systemd-analyze blame                # 느린 unit
systemd-analyze critical-chain
systemd-analyze plot > boot.svg

# kernel 부팅 시간
dmesg --time-format=delta | tail -30
```

---

## 11. 부팅 문제 디버그

### 11.1 emergency / rescue
GRUB 에서 cmdline 에 `systemd.unit=emergency.target` 추가.

```
systemd.unit=rescue.target           # minimal services
systemd.unit=emergency.target         # only root shell
init=/bin/bash                        # 가장 minimal
```

### 11.2 자주 검사
- kernel cmdline 에 잘못된 root UUID
- initramfs 가 module 누락
- fstab 잘못된 UUID
- LUKS 비밀번호 prompt
- network mount block

### 11.3 console / serial
serial console 활성 — VM / 원격 디버그.

```
GRUB_CMDLINE_LINUX_DEFAULT="console=ttyS0,115200"
```

---

## 12. fsck

부팅 시 자동:
```
- 마지막 mount 부터 N 회 mount 또는 N 일 후 → fsck
- /etc/fstab 의 passno 0/1/2
- /forcefsck 파일 → 한 번 강제
```

---

## 13. Secure Boot

```
UEFI 가 signed binary 만 부팅 허용
chain: shim (Microsoft 서명) → GRUB (vendor 서명) → kernel (vendor 서명)
```

기본 활성. custom kernel 시 직접 서명 필요.

```bash
mokutil --sb-state
```

---

## 14. Recovery / Live USB

- Live USB (Ubuntu, RHEL ISO) 부팅 → chroot 로 host 수리
- `arch-chroot /mnt`
- 패스워드 재설정, GRUB 복구, 파일 복구

```bash
# 패스워드 재설정
boot → init=/bin/bash
mount -o remount,rw /
passwd root
sync
reboot
```

자주 쓰는 SOS.

---

## 15. cloud-init (클라우드)

```
EC2 / GCE / Azure → user-data 스크립트 실행
초기 패스워드 / SSH key / 패키지 / 파일 설정
```

```bash
sudo cloud-init status
cat /var/log/cloud-init.log
sudo cloud-init clean       # 재실행
```

---

## 16. 컨테이너의 "부팅"

container 는 부팅 없음 — process 즉시 시작.
**PID 1** 의 책임 (signal / reap).

자세히 → [[../process/orphan-zombie#10-컨테이너의-pid-1-함정]]

---

## 17. Cold vs Warm Boot

- **Cold** — 전원 OFF → ON
- **Warm** — reboot (kernel reload)
- **kexec** — kernel 안에서 새 kernel 부팅 (BIOS 우회 — 빠름)

```bash
sudo kexec -l /boot/vmlinuz --initrd=/boot/initrd.img --reuse-cmdline
sudo kexec -e
```

서버 reboot 시간 단축.

---

## 18. 함정

### 18.1 GRUB update 누락
새 kernel 후 grub-update 안 하면 부팅 X.

### 18.2 root UUID 변경
fstab + GRUB cmdline 동기화.

### 18.3 initramfs 누락 module
드라이브 못 찾음 → kernel panic.

### 18.4 Secure Boot + custom kernel
mok 서명 또는 비활성.

### 18.5 LUKS prompt 누락
console attached 안 됨 — server 부팅 정지.

### 18.6 부팅 너무 빠름
keyboard interrupt 어려움 — GRUB timeout 늘리기.

### 18.7 hostname change
`/etc/hostname` 만 변경 → reboot.

### 18.8 swap UUID 변경
fstab의 옛 UUID → hibernate / swap 실패.

---

## 19. 학습 자료

- **systemd-analyze**
- **The Linux Programming Interface** Ch. 36
- **Arch wiki — Boot process**
- **dmesg** + **journalctl -b**

---

## 20. 관련

- [[systemd]]
- [[kernel-architecture]]
- [[linux]] — Linux hub
