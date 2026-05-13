---
title: "Shell / 명령 — bash / zsh / 자주 쓰는 도구"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T16:25:00+09:00
tags:
  - operating-system
  - linux
  - shell
---

# Shell / 명령

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | bash / 핵심 도구 |

**[[linux|↑ Linux hub]]**

---

## 1. Shell 종류

| Shell | 특징 |
| --- | --- |
| **bash** | GNU, 기본 |
| **zsh** | 풍부한 기능, oh-my-zsh / starship |
| **fish** | autosuggest, 직관 |
| **dash** | 작고 빠름 (`/bin/sh` on Ubuntu) |
| **ksh** | UNIX 표준 |
| **tcsh** | C shell 변형 |
| **busybox sh** | 임베디드 |

```bash
echo $SHELL
chsh -s /bin/zsh
```

---

## 2. 디렉토리 / 파일

```bash
pwd
cd dir / cd .. / cd -
ls -lha                       # 모두 + human size
ls -lhSr                       # size sorted
tree
find /path -name "*.log" -type f -mtime -7
find /path -size +100M
find /path -exec rm {} +

mkdir -p a/b/c
rmdir / rm -rf
cp -r a b
mv a b
ln -s target link              # symlink

stat file
file file
```

---

## 3. 보기 / 텍스트

```bash
cat / less / more
head -n 50 / tail -n 50
tail -f file.log
wc -l file
sort / uniq -c
cut -d',' -f1,3
paste a b                      # column join

grep "pattern" file
grep -rni "word" .
grep -E "(a|b)" file
grep -v "exclude"
grep -A3 -B3 "around"

sed 's/old/new/g' file
sed -i 's/old/new/g' file       # in place
sed -n '10,20p' file            # 줄 10-20

awk '{print $1}' file
awk -F',' '$3 > 100 {print}' file
awk 'BEGIN{s=0} {s+=$1} END{print s}'
```

---

## 4. Pipe / Redirect

```bash
ls | wc -l
ls > out.txt                    # stdout
ls >> out.txt                    # append
ls 2> err.txt                    # stderr
ls > out 2>&1                    # 둘 다
ls &> out                        # 둘 다 (bash)
< input.txt                       # stdin
<<< "string"                      # here-string
<<EOF                             # heredoc
text
EOF
```

---

## 5. 변수 / 환경

```bash
NAME="alice"
echo "$NAME"
echo "${NAME}_suffix"
echo "${NAME:-default}"          # null 이면 default
echo "${NAME:?error}"             # null 이면 에러

export PATH="$HOME/bin:$PATH"
unset NAME
env                               # 모든 env
declare -p NAME                    # 정보

readonly NAME
local x=1                          # function 안
```

---

## 6. 조건 / 반복

```bash
if [ -f file ]; then ...; fi
if [[ "$x" == "abc" ]]; then ...; fi
[ -d dir ] && cmd || echo nope

for f in *.txt; do echo "$f"; done
for i in {1..10}; do ...; done
for i in $(seq 1 10); do ...; done
while read line; do ...; done < file
case "$x" in a) ...;; b) ...;; *) ...;; esac
```

### 6.1 비교

| Test | 의미 |
| --- | --- |
| `-f file` | 파일 |
| `-d dir` | 디렉토리 |
| `-e exists` | 존재 |
| `-x` | 실행 |
| `-r/-w` | read/write |
| `-z` | 빈 문자열 |
| `-n` | 비어있지 X |
| `-eq/-ne/-lt/-gt` | 숫자 |
| `=`/`!=` | 문자열 |

---

## 7. Function

```bash
greet() {
  local name="$1"
  echo "Hi $name"
  return 0
}
greet "alice"
```

---

## 8. 자주 쓰는 도구

### 8.1 텍스트 처리

```bash
tr ' ' '\n'                       # 치환
tr -d '\r'                         # 삭제
fold -w 80                         # 줄 자르기
column -t                          # 컬럼 정렬
xargs                              # 인자로 변환
xargs -P 4 -I{} cmd {}             # 병렬

jq '.users[].name' file.json
yq '.spec.containers[0].image' k.yaml
```

### 8.2 비교 / merge

```bash
diff a b
diff -u a b                        # unified
sdiff a b
comm a b                            # 정렬된 파일 비교
patch < diff
vimdiff a b
```

### 8.3 파일 처리

```bash
tar cvzf out.tar.gz dir
tar xvzf out.tar.gz
tar xf file.tar.zst                # zstd
unzip / zip
gzip / gunzip
xz / unxz
zstd / unzstd

split -b 100M file part-
cat part-* > file
md5sum / sha256sum
shred -uvz secret                  # 안전 삭제
```

### 8.4 네트워크 / 다운로드

```bash
curl -fsSL https://...
curl -X POST -d '{"k":"v"}' -H 'Content-Type: application/json' url
wget https://...
wget -m https://site.com           # mirror

scp file user@host:/path
rsync -avzP src/ dst/              # 가장 강력
```

자세히 → [[networking]]

### 8.5 process

```bash
ps aux | grep nginx
pgrep -f nginx
pkill -HUP nginx
killall nginx
top / htop / btop
pidof nginx
fuser file                          # 파일 잡은 process
lsof / lsof -i :8080 / lsof -p $PID
```

### 8.6 file / dir 비교

```bash
md5sum *
sha256sum -c sums.txt
```

---

## 9. Job control

```bash
cmd &                               # 백그라운드
jobs
fg %1
bg %1
disown                              # job table 제거
nohup cmd &                         # logout 후에도 살아 있음
Ctrl-Z                              # 일시 정지

# screen / tmux 로 detached session
tmux new -s mysession
tmux attach -t mysession
Ctrl-b d                            # detach
```

---

## 10. History

```bash
history
!!                                  # 마지막 명령
!42                                  # 42번째
!grep                                # 마지막 grep
Ctrl-R                              # 인터랙티브 검색
HISTSIZE=10000
```

`~/.bash_history` / `~/.zsh_history`.

---

## 11. Quoting

```bash
echo 'literal $HOME'                # single — 그대로
echo "expand $HOME"                  # double — 변수 + escape
echo `cmd`                           # 옛 command sub
echo $(cmd)                          # 현대
echo $(( 1 + 2 ))                    # 산술
```

⚠️ Word splitting + glob → 항상 변수에 `"$var"` 사용.

---

## 12. globbing

```bash
*.txt                                # 모든 .txt
?.txt                                # 한 글자
[abc].txt
{a,b,c}.txt                          # brace
**/foo                               # globstar (shopt -s globstar)
```

---

## 13. 자주 보는 함정

### 13.1 `cd` 실패 후 계속
```bash
cd dir && rm -rf *                    # cd 실패 시 / 의 rm 위험
```
→ `set -e` 또는 `||exit`.

### 13.2 `set -euo pipefail` (bash strict)
```bash
set -e          # 에러 시 종료
set -u          # 미설정 변수 에러
set -o pipefail # pipe 의 어느 명령 실패 시
```

### 13.3 `IFS` 변경
default = whitespace. `IFS=','` 등으로 변경 시 다른 동작.

### 13.4 `eval` 사용
입력 신뢰 X — injection 위험.

### 13.5 `rm -rf $VAR/...`
$VAR 가 빈 값 → `rm -rf /...`. quote + check.

### 13.6 `echo $line` (quote 없음)
공백 / 글로빙. `echo "$line"`.

### 13.7 cd 의 후속
script 안의 cd 는 그 script 만 — 호출자엔 X.

### 13.8 `[ a == b ]`
일부 shell 에선 syntax error. `=` 또는 `[[ ... ]]` (bash/zsh).

---

## 14. 좋은 사용 권장

```bash
#!/usr/bin/env bash
set -euo pipefail

# 변수
readonly OUT_DIR="${1:-/tmp}"
declare -r FILE="$OUT_DIR/log"

# function
log() { echo "[$(date +%T)] $*" >&2; }

# trap
trap 'rm -f /tmp/$$.*' EXIT INT TERM
```

shellcheck:
```bash
shellcheck script.sh
```

---

## 15. zsh 의 추가

```zsh
# powerline / starship
# autosuggest (oh-my-zsh)
# completion
# globalias
# extended globbing
```

---

## 16. 학습 자료

- **Bash Manual** — gnu.org/software/bash/manual/
- **ABS (Advanced Bash Scripting Guide)**
- **Pure bash bible** (github)
- **shellcheck.net**
- **Linux Command Line and Shell Scripting Bible**

---

## 17. 관련

- [[file-permissions]]
- [[../syscall/syscall]] — shell 의 syscall
- [[linux]] — Linux hub
