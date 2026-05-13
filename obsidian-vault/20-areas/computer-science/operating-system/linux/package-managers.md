---
title: "Linux Package Managers — apt / dnf / pacman / apk / snap / flatpak"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:35:00+09:00
tags:
  - operating-system
  - linux
  - package
---

# Linux Package Managers

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 주요 매니저 + 명령 |

**[[linux|↑ Linux hub]]**

---

## 1. APT (Debian / Ubuntu)

```bash
sudo apt update                       # 인덱스 갱신
sudo apt upgrade                       # 업그레이드
sudo apt full-upgrade                  # + 의존성 추가/제거
sudo apt install nginx
sudo apt install -y nginx redis-server
sudo apt remove nginx
sudo apt purge nginx                   # + config 제거
sudo apt autoremove                    # 자동 의존 제거

apt search nginx
apt show nginx
apt list --installed
apt list --upgradable
apt depends nginx
apt rdepends nginx
apt-mark hold nginx                    # update 잠금
apt-mark unhold nginx

# 저장소
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/

sudo add-apt-repository ppa:user/repo
sudo apt-key list                      # deprecated → /etc/apt/keyrings/

# 캐시 / 정리
apt-cache search nginx
sudo apt clean
sudo apt autoclean
```

### 1.1 dpkg (low-level)

```bash
sudo dpkg -i package.deb
sudo dpkg -r nginx
sudo dpkg -P nginx                     # purge
dpkg -l                                # 설치 목록
dpkg -L nginx                           # 파일 목록
dpkg -S /etc/nginx/nginx.conf           # 파일 → 패키지
dpkg --configure -a                     # 실패 복구
```

---

## 2. DNF / YUM (RHEL / Fedora)

```bash
sudo dnf update                        # = upgrade
sudo dnf install nginx
sudo dnf remove nginx
sudo dnf autoremove

dnf search nginx
dnf info nginx
dnf list installed
dnf list updates
dnf history                            # 변경 이력
dnf history undo 23
dnf provides /usr/bin/whatever          # 파일 → 패키지
dnf repolist
dnf module list                         # 모듈

# 저장소
ls /etc/yum.repos.d/
sudo dnf config-manager --add-repo URL
sudo dnf config-manager --enable repo-name
```

### 2.1 rpm (low-level)

```bash
sudo rpm -ivh package.rpm
sudo rpm -e nginx
rpm -qa                                # 설치 목록
rpm -ql nginx
rpm -qf /etc/nginx/nginx.conf
rpm -V nginx                            # 무결성 검증
```

---

## 3. pacman (Arch)

```bash
sudo pacman -Syu                        # update + upgrade
sudo pacman -S nginx                    # install
sudo pacman -R nginx                    # remove
sudo pacman -Rns nginx                   # + 의존성 + config

pacman -Ss nginx                        # search remote
pacman -Qs nginx                         # search local
pacman -Qi nginx                         # info
pacman -Ql nginx                         # files
pacman -Qo /usr/bin/whatever             # file → package
pacman -Qdt                              # orphan packages
pacman -Sc                               # clean cache

# AUR (yay / paru)
yay -S aur-package
```

---

## 4. zypper (openSUSE)

```bash
sudo zypper refresh
sudo zypper update
sudo zypper install nginx
sudo zypper remove nginx
zypper search nginx
zypper info nginx
zypper lr                              # repo list
```

---

## 5. apk (Alpine)

```bash
apk update
apk upgrade
apk add nginx
apk del nginx
apk info -L nginx                       # 파일 목록
apk info -W /path                        # 파일 → 패키지
```

Dockerfile 표준:
```dockerfile
FROM alpine:3.20
RUN apk add --no-cache nginx
```

`--no-cache` = update + install + index 삭제 한 layer.

---

## 6. Universal — Snap / Flatpak / AppImage

### 6.1 Snap (Canonical)
```bash
sudo snap install code
snap list
snap refresh
sudo snap remove code
```

- container 비슷 격리
- universal 단일 binary
- 자동 update
- 단점: 큰 size, mount overhead

### 6.2 Flatpak
```bash
flatpak install flathub org.gimp.GIMP
flatpak list
flatpak update
flatpak run org.gimp.GIMP
```

- 데스크탑 응용
- sandbox (bubblewrap)

### 6.3 AppImage
```
single binary — 다운로드 + 실행
chmod +x app.AppImage
./app.AppImage
```

- 가장 단순
- update / 통합 약함

---

## 7. NixOS / Nix

```bash
nix-env -i hello              # imperative
nix-env -e hello
nix-env -q                     # 설치 목록

# declarative
# /etc/nixos/configuration.nix
{ environment.systemPackages = with pkgs; [ vim git nginx ]; }
sudo nixos-rebuild switch

# flake
nix shell nixpkgs#hello
```

reproducible / rollback.

---

## 8. Container 안의 패키지

```bash
# 작은 image 유지
RUN apt-get update && apt-get install -y --no-install-recommends ... \
    && rm -rf /var/lib/apt/lists/*

RUN apk add --no-cache nginx
```

multi-stage build 로 build dep 분리.

---

## 9. Language-specific

| Lang | 매니저 |
| --- | --- |
| Python | pip, poetry, uv, conda |
| Node | npm, yarn, pnpm |
| Ruby | gem, bundler |
| Rust | cargo |
| Go | go get / go mod |
| Java | maven, gradle |
| .NET | nuget |

OS 패키지와 분리 — `/opt`, `~/.local`, `~/.cache` 등.

---

## 10. 보안

```bash
# CVE 검사
apt list --upgradable | grep -i security
sudo unattended-upgrades              # 자동 보안 update

# 패키지 검증
debsums                                # debian
rpm -Va                                # redhat
```

- 공식 repo 만 신뢰
- 외부 PPA / repo 신중
- GPG signature 검증

---

## 11. 자주 보는 문제

### 11.1 broken dependency

```bash
sudo apt --fix-broken install
sudo dpkg --configure -a
sudo dnf check
```

### 11.2 lock 잡힘

```
Could not get lock /var/lib/dpkg/lock
```
다른 apt 가 실행 중. `lsof /var/lib/dpkg/lock` 확인.

### 11.3 GPG 키 없음

```bash
# 옛
sudo apt-key add key.gpg
# 신
sudo wget -O /etc/apt/keyrings/key.gpg https://...
```

### 11.4 source 미러 느림

```
/etc/apt/sources.list 의 미러 변경
한국: kr.archive.ubuntu.com
```

### 11.5 hold 한 패키지 update 안 됨

```bash
apt-mark showhold
sudo apt-mark unhold nginx
```

---

## 12. 함정

### 12.1 `apt-get install` 의존 누락
`-y` + production 자동 update 위험. 명시적 관리.

### 12.2 PPA 추가 후 OS 업그레이드 깨짐
distro upgrade 전 PPA disable.

### 12.3 dpkg / rpm 직접 install
의존성 자동 해결 안 됨. apt/dnf 사용.

### 12.4 snap 의 디스크 사용량
오래된 버전 자동 정리 안 함. `sudo snap set system refresh.retain=2`.

### 12.5 alpine + glibc binary
musl 차이로 실패. `gcompat` 또는 다른 base.

### 12.6 `apt upgrade` vs `apt full-upgrade`
full-upgrade 가 패키지 제거 가능 — distro upgrade 외엔 upgrade.

### 12.7 cache 폭증
`/var/cache/apt`, `/var/cache/dnf`. 정기 clean.

### 12.8 패키지 다운그레이드
대부분 매니저 지원 X — 정확한 버전 명시 install (`apt install nginx=1.18.0-0ubuntu1`).

---

## 13. 학습 자료

- 각 매니저 man page
- Debian / Ubuntu / Fedora / Arch wiki
- **Linux Package Management** — DigitalOcean 가이드

---

## 14. 관련

- [[distributions]]
- [[../security/security]]
- [[linux]] — Linux hub
