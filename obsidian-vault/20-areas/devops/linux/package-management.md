---
title: "Package management — apt / yum / dnf"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:55:00+09:00
tags: [devops, linux, package]
---

# Package management — apt / yum / dnf

**[[linux|↑ linux]]**

---

## 1. distro 별

| Distro | tool | format |
| --- | --- | --- |
| Debian / Ubuntu | apt / apt-get / dpkg | .deb |
| RHEL / CentOS 7 | yum / rpm | .rpm |
| RHEL 8+ / Fedora / Rocky | dnf / rpm | .rpm |
| Alpine | apk | .apk |
| Arch | pacman | .pkg.tar.zst |
| openSUSE | zypper | .rpm |

---

## 2. apt (Debian / Ubuntu)

```bash
sudo apt update                       # repo index 갱신 (★ 먼저)
sudo apt upgrade                      # 설치된 것 업그레이드
sudo apt full-upgrade                  # kernel / 의존성 포함

sudo apt install nginx
sudo apt install nginx=1.22.0-1ubuntu1  # 특정 버전
sudo apt remove nginx                 # 설정 유지
sudo apt purge nginx                  # 설정도 삭제
sudo apt autoremove                   # 의존성 정리

apt search nginx
apt show nginx
apt list --installed
apt list --upgradable

# repo 추가
sudo add-apt-repository ppa:nginx/stable
# 또는 manual:
echo "deb https://nginx.org/packages/ubuntu/ jammy nginx" | \
    sudo tee /etc/apt/sources.list.d/nginx.list
curl -fsSL https://nginx.org/keys/nginx_signing.key | \
    sudo gpg --dearmor -o /etc/apt/keyrings/nginx.gpg
sudo apt update
```

---

## 3. dnf (RHEL 8+ / Rocky / Fedora)

```bash
sudo dnf check-update
sudo dnf upgrade

sudo dnf install nginx
sudo dnf remove nginx
sudo dnf autoremove

dnf search nginx
dnf info nginx
dnf list installed
dnf history                          # 작업 이력
sudo dnf history undo 42             # 특정 작업 되돌리기

# repo 추가
sudo dnf config-manager --add-repo https://nginx.org/packages/centos/9/x86_64/
sudo rpm --import https://nginx.org/keys/nginx_signing.key

# group
dnf groupinstall "Development Tools"
```

---

## 4. yum (RHEL 7 — legacy)

```bash
sudo yum update
sudo yum install nginx
sudo yum remove nginx
```

→ RHEL 8+ 에서 yum 은 dnf 의 alias.

---

## 5. apk (Alpine)

```bash
apk update
apk upgrade
apk add nginx
apk add --no-cache nginx              # cache 안 남김 (container)
apk del nginx
apk info -a nginx
apk search nginx
```

→ Docker 에서 `apk add --no-cache` 이미지 작게.

---

## 6. dpkg / rpm (low-level)

```bash
# dpkg
dpkg -l                              # 설치된 패키지
dpkg -L nginx                        # nginx 가 설치한 파일
dpkg -S /usr/sbin/nginx              # 파일이 어느 패키지?
dpkg -i nginx.deb                    # local file 설치

# rpm
rpm -qa                              # 전체
rpm -ql nginx                        # 설치 파일
rpm -qf /usr/sbin/nginx              # 파일 → 패키지
rpm -i nginx.rpm                     # 설치
rpm -e nginx                         # 제거
```

---

## 7. 의존성 / repo / GPG

```
- 패키지 = 메타데이터 + 파일
- repo = 패키지 모음 (HTTP/HTTPS)
- GPG key = 서명 검증 (★ 보안)
- 의존성 = 다른 패키지 / 버전
```

→ **반드시 GPG key 검증**. 검증 안 된 repo = 공급망 공격 위험.

---

## 8. unattended-upgrade (자동 보안 패치)

```bash
# Ubuntu
sudo apt install unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades

# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
```

→ 보안 패치 자동. **production 서버 필수**.

---

## 9. snap / flatpak (universal)

```bash
# snap (Ubuntu)
sudo snap install firefox
snap list
snap refresh

# flatpak (Fedora 등)
flatpak install flathub org.mozilla.firefox
```

→ container 화된 패키지, 격리.

---

## 10. version pinning (안정성)

```bash
# apt
apt-mark hold nginx                  # upgrade 제외
apt-mark unhold nginx

# dnf
echo "exclude=nginx*" >> /etc/dnf/dnf.conf

# 또는 install 시 versionlock plugin
sudo dnf install python3-dnf-plugin-versionlock
sudo dnf versionlock add nginx
```

---

## 11. 함정

1. **`apt upgrade` 만 — index update 안 함** → 오래된 버전 설치.
2. **`curl | sh`** — script 검증 X (보안).
3. **GPG key 검증 안 함** — 위조 repo 위험.
4. **production 에서 자동 major upgrade** — breaking change.
5. **disk full** — `/var/cache/apt`, `/var/lib/dnf` 청소: `apt clean` / `dnf clean all`.
6. **dependency hell** — 너무 많은 PPA → 깨지는 의존성.

---

## 12. 관련

- [[linux|↑ linux]]
- [[systemd]]
- [[../docker/docker|↗ docker (apk add --no-cache)]]
