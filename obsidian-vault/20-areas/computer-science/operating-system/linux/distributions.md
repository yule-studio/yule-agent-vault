---
title: "Linux 배포판 — Debian / Ubuntu / RHEL / Arch / Alpine"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:05:00+09:00
tags:
  - operating-system
  - linux
  - distribution
---

# Linux Distributions

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 주요 배포판 비교 |

**[[linux|↑ Linux hub]]**

---

## 1. 배포판 = Kernel + Userland + Packaging

같은 Linux kernel 위에 다른 userland (glibc / musl / busybox), 다른 init, 다른 패키지 매니저, 다른 정책.

---

## 2. 주요 계열

### 2.1 Debian 계열
| 배포 | 특징 |
| --- | --- |
| **Debian** | 가장 보수적, 무료 |
| **Ubuntu** | Canonical, LTS 5/10년 |
| **Linux Mint** | 데스크탑 |
| **Pop!_OS** | System76 |
| **Kali** | 보안 |
| **Raspberry Pi OS** | RPi |

패키지: `dpkg` / `apt`.

### 2.2 Red Hat 계열
| 배포 | 특징 |
| --- | --- |
| **RHEL** | 상용, 엔터프라이즈 표준 |
| **CentOS** (옛) | RHEL 무료 → CentOS Stream (rolling) |
| **AlmaLinux** | RHEL 무료 대체 |
| **Rocky Linux** | RHEL 무료 대체 |
| **Fedora** | Red Hat 의 testing 배포 |
| **Amazon Linux** | AWS |
| **Oracle Linux** | Oracle |

패키지: `rpm` / `dnf` (옛 yum).

### 2.3 Arch 계열
| 배포 | 특징 |
| --- | --- |
| **Arch Linux** | rolling release, minimal |
| **Manjaro** | 사용자 친화 |
| **EndeavourOS** | Arch + 친화 |

패키지: `pacman` + **AUR** (커뮤니티).

### 2.4 SUSE 계열
| 배포 | 특징 |
| --- | --- |
| **openSUSE** (Leap/Tumbleweed) | 커뮤니티 |
| **SLES** | SUSE Enterprise |

패키지: `rpm` / `zypper`.

### 2.5 Independent
| 배포 | 특징 |
| --- | --- |
| **Alpine** | musl + busybox, 작음 (5 MB) — Docker 표준 |
| **NixOS** | declarative, atomic |
| **Gentoo** | source-based |
| **Slackware** | 옛 |
| **Void** | runit init |
| **CRUX / LFS** | 학습용 |

### 2.6 컨테이너 / 임베디드 특화
| 배포 | 특징 |
| --- | --- |
| **Alpine Linux** | 컨테이너 표준 (musl) |
| **distroless** (Google) | base 만 — bash 없음 |
| **scratch** | 완전 비어있음 |
| **CoreOS / Flatcar** | container-native (immutable, auto-update) |
| **Bottlerocket** (AWS) | K8s host OS |
| **Talos Linux** | K8s 전용 |
| **Yocto / Buildroot** | 임베디드 build |
| **OpenWrt** | 라우터 |
| **Android (AOSP)** | 모바일 (Linux 기반) |

---

## 3. 패키지 매니저 비교

| 매니저 | 배포판 | 명령 |
| --- | --- | --- |
| **dpkg/apt** | Debian/Ubuntu | `apt install pkg` |
| **rpm/dnf** | RHEL/Fedora | `dnf install pkg` |
| **pacman** | Arch | `pacman -S pkg` |
| **zypper** | openSUSE | `zypper install pkg` |
| **apk** | Alpine | `apk add pkg` |
| **xbps** | Void | `xbps-install pkg` |
| **nix** | NixOS | `nix-env -i pkg` 또는 declarative |
| **emerge** | Gentoo | `emerge pkg` |
| **snap** | universal | `snap install pkg` |
| **flatpak** | universal | `flatpak install pkg` |
| **AppImage** | universal | run 직접 |

자세히 → [[package-managers]]

---

## 4. Release 주기

| | 안정성 | 빈도 |
| --- | --- | --- |
| **LTS** (Ubuntu / RHEL) | ★★★★★ | 2년 |
| **Stable** (Debian / Fedora) | ★★★★ | 6개월 ~ 2년 |
| **Rolling** (Arch / Tumbleweed) | ★★★ | 매일 |
| **CentOS Stream** | ★★★ | continuous |

운영 = LTS / Stable.
데스크탑 / 개발 = rolling 가능.

---

## 5. systemd vs 다른 init

| Init | 사용 배포 |
| --- | --- |
| **systemd** | 대부분 (Debian, Ubuntu, RHEL, Fedora, Arch, openSUSE) |
| **OpenRC** | Gentoo, Alpine |
| **runit** | Void |
| **s6** | Adelie |
| **busybox init** | 임베디드 |
| **SysV init (옛)** | 옛 배포판 |

자세히 → [[systemd]]

---

## 6. /etc/os-release

```bash
cat /etc/os-release
# NAME="Ubuntu"
# VERSION_ID="24.04"
# PRETTY_NAME="Ubuntu 24.04 LTS"
# ID=ubuntu
# ID_LIKE=debian
# VERSION_CODENAME=noble
```

스크립트에서 배포판 분기:
```bash
. /etc/os-release
case "$ID" in
  ubuntu|debian) apt install -y ... ;;
  fedora|rhel|centos|rocky|almalinux) dnf install -y ... ;;
  arch) pacman -S --noconfirm ... ;;
  alpine) apk add ... ;;
esac
```

---

## 7. Alpine — 컨테이너 표준

```
+ 작음 (5 MB base)
+ musl libc (작고 빠름)
+ busybox (모든 shell util)

- glibc 호환 X (일부 binary 동작 X)
- DNS 처리 차이 (musl)
- 일부 보안 모듈 미지원
```

→ Docker base 의 표준. 단, 호환 이슈 검토.

---

## 8. distroless / scratch

```dockerfile
FROM gcr.io/distroless/static-debian12
COPY myapp /
ENTRYPOINT ["/myapp"]
```

- shell 없음
- 패키지 매니저 없음
- 보안 / 크기 ↑
- 디버그 어려움 (debug variant 있음)

---

## 9. NixOS — Declarative

```nix
{ config, pkgs, ... }:
{
  services.nginx.enable = true;
  users.users.alice = {
    isNormalUser = true;
    extraGroups = [ "wheel" ];
  };
}
```

- 전체 시스템 = 단일 설정 파일
- atomic rollback
- 재현성

CI / lab / 인프라 IaC.

---

## 10. 선택 가이드

| 시나리오 | 추천 |
| --- | --- |
| 일반 서버 | Ubuntu LTS / Debian / Rocky / AlmaLinux |
| Enterprise + 지원 | RHEL / SLES |
| 컨테이너 base | Alpine / distroless |
| 데스크탑 (입문) | Ubuntu / Mint / Pop |
| 데스크탑 (커스텀) | Arch / openSUSE Tumbleweed / Fedora |
| 보안 / 침투 | Kali / Parrot |
| 임베디드 / IoT | Buildroot / Yocto / OpenWrt |
| K8s host | Flatcar / Bottlerocket / Talos |
| Cloud (AWS) | Amazon Linux (또는 Ubuntu) |
| 학습 / hardcore | Arch / Gentoo / LFS |

---

## 11. 한국 사용자

- Ubuntu 한국 사용자 ↑
- RHEL / CentOS 도 기업
- 한국어 입력 / 폰트는 모든 데스크탑에 가능

`im-config` (Ubuntu) / `ibus / fcitx` 로 한국어.

---

## 12. 함정

### 12.1 Alpine 의 glibc 의존 응용
"works on my machine" 후 production fail. node-gyp 등 native build.

### 12.2 LTS 의 옛 패키지
Ubuntu 24.04 의 docker / kubernetes 는 별도 repo / snap.

### 12.3 Fedora / RHEL Stream 의 변화
RHEL = downstream 옛 → CentOS Stream = upstream. 운영 영향 검토.

### 12.4 Arch rolling 의 break
정기 update 필수. 한 달 두면 종속성 폭발.

### 12.5 NixOS 의 학습 곡선
Nix language 학습.

### 12.6 distroless 디버그
debug variant 또는 ephemeral container.

### 12.7 라이선스 (RHEL)
재배포 / 클라우드 사용 시 라이선스 검토. AlmaLinux / Rocky 대안.

---

## 13. 학습 자료

- **DistroWatch** — distrowatch.com
- **각 배포판 공식 docs**
- **Linux From Scratch** (LFS) — 깊은 이해
- **Arch Wiki** — 모든 Linux 사용자에게 좋은 reference

---

## 14. 관련

- [[package-managers]]
- [[systemd]]
- [[../virtualization/container]]
- [[linux]] — Linux hub
