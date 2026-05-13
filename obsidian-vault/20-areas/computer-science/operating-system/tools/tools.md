---
title: "OS 도구 (Hub)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T18:00:00+09:00
tags:
  - operating-system
  - tools
  - hub
---

# OS 도구 (Hub)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | hub + 도구 카탈로그 |

**[[../operating-system|↑ OS hub]]**

---

## 1. 카테고리

| 카테고리 | 도구 |
| --- | --- |
| **모니터링** | top / htop / btop / glances / atop |
| **시스템 통계** | vmstat / iostat / mpstat / sar |
| **프로세스** | ps / pidstat / pgrep |
| **메모리** | free / smem / pmap |
| **디스크** | df / du / iotop / iostat / lsblk |
| **네트워크** | ip / ss / netstat / tcpdump / nload / nethogs |
| **트레이싱** | strace / ltrace / dtruss |
| **프로파일링** | perf / valgrind / gdb |
| **eBPF** | bcc-tools / bpftrace / libbpf-tools |
| **로그** | journalctl / dmesg / logrotate |
| **편집** | vim / nano / emacs |
| **검색** | grep / ag / ripgrep / fd / fzf |
| **diff / patch** | diff / vimdiff / patch / delta |
| **파일** | ls / find / lsof / tree / file |
| **압축** | tar / gzip / zstd / xz / 7z |
| **전송** | scp / rsync / curl / wget |
| **container** | docker / podman / nerdctl / runc |
| **k8s** | kubectl / k9s / stern / kustomize |
| **terminal** | tmux / screen / zellij |

---

## 2. 빠른 cheatsheet

자주 보는 도구는 별도 노트:
- [[gdb]] — debugger
- [[lsof]] — list open files
- [[perf]] — performance
- [[../syscall/strace]] — syscall trace
- [[../linux/ebpf]] — eBPF
- [[../linux/monitoring]] — 시스템 모니터링
- [[../linux/strace-perf]] — strace / perf

---

## 3. 모르면 시작 — `tldr`

```bash
sudo apt install tldr
tldr tar                                  # 짧은 예시
tldr ls
tldr ssh
```

`man` 보다 친화적인 example 모음.

---

## 4. file / which / type

```bash
file myapp                                 # binary 종류
file -i myapp                               # MIME

which ls                                    # PATH 의 위치
type ls                                     # alias / function 포함
type -a ls
```

---

## 5. lsof — 열린 파일

```bash
sudo lsof -p $PID
sudo lsof -u alice
sudo lsof /var/log/syslog                  # 어떤 process 가
sudo lsof -i :8080                          # 포트
sudo lsof -i tcp:443
sudo lsof | grep deleted                    # 삭제된 file 잡고 있는 process (디스크 누수)
```

자세히 → [[lsof]]

---

## 6. tree

```bash
sudo apt install tree
tree -L 2
tree -I 'node_modules|.git'
tree -d                                     # 디렉토리만
```

---

## 7. ripgrep / ag (faster grep)

```bash
sudo apt install ripgrep
rg "pattern" .
rg -i "case insensitive"
rg -t py "pattern"                          # type 한정
rg --files | rg test
```

`ag` (silver searcher) 도 비슷.

---

## 8. fd — fast find

```bash
sudo apt install fd-find
fd pattern                                  # 빠른 find
fd -e py                                    # extension
fd -t d                                     # type
fd -H hidden                                # hidden
```

---

## 9. fzf — fuzzy finder

```bash
sudo apt install fzf

vim $(fzf)                                  # 파일 선택
history | fzf
ps aux | fzf | awk '{print $2}'

# shell 통합
Ctrl-R                                      # history 검색 (강력)
Ctrl-T                                      # 파일 선택
Alt-C                                       # 디렉토리
```

---

## 10. jq / yq — JSON / YAML

```bash
jq '.users[].name' file.json
jq -r '.[] | select(.age > 30)' file.json
echo '{"a":1}' | jq '.a'

yq '.spec.replicas' deployment.yaml
yq -i '.spec.replicas = 3' deployment.yaml
```

`jq` 가 사실상 표준. `yq` 도 jq syntax (또는 별도).

---

## 11. htop / btop

```bash
htop                                        # 더 좋은 top
btop                                        # 가장 예쁜
glances                                     # 종합

# 사용
F4 — filter
F5 — tree
F6 — sort
F9 — kill
F10 — quit
```

---

## 12. 터미널 멀티플렉서

### 12.1 tmux

```bash
tmux new -s mywork
tmux ls
tmux attach -t mywork

# 키 (default prefix Ctrl-b)
Ctrl-b c        # new window
Ctrl-b n / p    # next / prev
Ctrl-b "        # horizontal split
Ctrl-b %        # vertical split
Ctrl-b z        # zoom pane
Ctrl-b d        # detach
Ctrl-b ?        # help
```

### 12.2 screen / zellij

screen = 옛. zellij = 새 Rust 기반.

---

## 13. dd / hexdump

```bash
dd if=/dev/zero of=test bs=1M count=100        # 100 MB 파일
dd if=/dev/urandom of=random bs=1M count=10

# block 단위 복사 (주의 — destination 덮어씀)
dd if=image.iso of=/dev/sdX bs=4M status=progress

hexdump -C file | head
xxd file | head
```

---

## 14. 시스템 정보

```bash
uname -a
lsb_release -a
cat /etc/os-release
hostnamectl
neofetch / screenfetch / fastfetch

lscpu
lsmem
lspci
lsusb
lsblk
lshw

dmidecode                                   # BIOS / motherboard
```

---

## 15. apt / dnf / pacman

자세히 → [[../linux/package-managers]]

---

## 16. systemctl / journalctl

자세히 → [[../linux/systemd]], [[../linux/logging]]

---

## 17. cron / at

```bash
crontab -e

at now + 1 hour
> command
> Ctrl-D
atq
atrm 1
```

자세히 → [[../linux/cron-systemd-timer]]

---

## 18. NetCat / socat

```bash
nc -lv 8080                                 # listen
nc -zv host 22                              # 포트 확인
echo "GET / HTTP/1.0\r\n\r\n" | nc -q 1 host 80

socat - TCP:host:80
socat TCP-LISTEN:1234,fork TCP:host:22      # tunnel
socat -d -d - UNIX:/var/run/docker.sock     # Unix socket
```

`socat` = nc 의 강력 변형.

---

## 19. 네트워크 분석

```bash
ping -c 4 host
mtr -n host                                  # 추세 traceroute
tracepath host

dig +short example.com
dig +trace example.com

# curl 디버그
curl -v https://...
curl -o /dev/null -w '%{time_total}\n' https://...
```

자세히 → [[../linux/networking]]

---

## 20. 파일 / 디스크

```bash
du -sh /path                                # 크기
ncdu / dust                                  # interactive
df -h
df -i                                        # inode

duf                                          # 예쁜 df
broot                                         # 디렉토리 explorer
```

---

## 21. 빌드 / 컴파일

```bash
make / cmake / meson / bazel
gcc / clang / rustc / go
go build / cargo build / mvn / gradle
```

---

## 22. git / version control

```bash
git status / diff / log
git pull --rebase
git add -p / commit / push
git stash / pop
git rebase -i
gitk / git log --graph
lazygit / tig
```

---

## 23. K8s / Container

```bash
docker / podman / nerdctl / lima / orbstack
docker ps / images / logs / exec / build
kubectl / k9s / lens
helm / kustomize
kind / minikube / k3s

stern <pod>          # 여러 pod log
kail / kubectl-stern
```

---

## 24. Sysadmin essentials

```bash
sudo / su
visudo
last / lastb / w / who
chage / passwd
useradd / usermod / groupadd
chsh / chfn
```

---

## 25. 학습 도구

- **tldr** — 짧은 예시
- **explainshell.com** — 명령 분해 설명
- **shellcheck.net** — script 검증
- **regex101.com** — regex
- **devhints.io** — cheatsheet

---

## 26. 관련

- [[gdb]] (작성 시)
- [[lsof]] (작성 시)
- [[perf]] (작성 시)
- [[../linux/linux]]
- [[../operating-system|↑ OS hub]]
