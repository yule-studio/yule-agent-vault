---
title: "Shell basics — pipe / redirect / 변수"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T06:34:00+09:00
tags: [devops, linux, shell]
---

# Shell basics — pipe / redirect / 변수

**[[linux|↑ linux]]**

---

## 1. shell 종류

| shell | 특징 |
| --- | --- |
| **bash** | 기본 (대부분 linux) |
| **zsh** | macOS 기본, plugin 풍부 |
| **fish** | 사용자 친화, POSIX 비호환 |
| **sh** (dash) | minimal, alpine / debian 의 /bin/sh |

→ **script 는 bash**, **interactive 는 zsh** 권장.

---

## 2. 명령 흐름

```
$ command [options] [arguments]

$ ls -lh /var/log
  └─ cmd
       └─ option (-lh)
                └─ argument (/var/log)
```

---

## 3. redirect

```bash
cmd > file        # stdout → file (덮어쓰기)
cmd >> file       # stdout → file (append)
cmd 2> err.log    # stderr → file
cmd > out 2>&1    # stdout + stderr → out
cmd &> all        # 위와 동일 (bash 4+)
cmd < input.txt   # stdin ← file
cmd <<< "string"  # here-string
cmd <<EOF         # here-doc
multi line
EOF
```

→ **2>&1** 의 순서 중요: `>out 2>&1` 와 `2>&1 >out` 다름.

---

## 4. pipe

```bash
cat /var/log/syslog | grep ERROR | wc -l
       ↓ stdout            ↓ stdout       ↓ stdout
```

```bash
# 자주 쓰는 조합
ps aux | grep nginx | grep -v grep
ls -1 | sort | uniq -c | sort -rn | head
docker ps --format '{{.Names}}' | xargs docker logs --tail 10
```

---

## 5. 변수

```bash
NAME="alice"
echo "Hello $NAME"            # 보간
echo 'Hello $NAME'            # 그대로 (single quote)
echo "Hello ${NAME}_user"     # boundary 명시

# 환경변수
export DB_URL=jdbc:...        # 자식 프로세스에 전파
unset DB_URL

# 명령 substitution
NOW=$(date +%Y-%m-%d)
SIZE=`du -sh /var | cut -f1`  # backtick (old style)

# default value
echo "${NAME:-anonymous}"     # NAME 비어있으면 anonymous
echo "${NAME:?required}"      # 비어있으면 error
```

---

## 6. 조건 / 반복

```bash
# if
if [ -f /etc/passwd ]; then
    echo "exists"
elif [ -d /tmp ]; then
    echo "tmp dir"
else
    echo "neither"
fi

# 비교 연산자
[ "$a" = "$b" ]          # 문자열
[ "$a" -eq "$b" ]        # 숫자 -eq -lt -gt -le -ge -ne
[ -z "$a" ]              # 비어있음
[ -n "$a" ]              # 비어있지 않음
[ -f file ]              # 파일 존재
[ -d dir ]               # 디렉터리 존재
[ -x cmd ]               # 실행 가능

# [[ ]] — bash 확장
[[ "$str" =~ ^[0-9]+$ ]] # 정규식
[[ "$a" == *bar* ]]      # glob

# for
for f in *.log; do
    gzip "$f"
done

for i in {1..10}; do echo $i; done

# while
while read line; do
    echo "$line"
done < input.txt

# until
until ping -c1 host; do sleep 1; done
```

---

## 7. && || ;

```bash
cmd1 && cmd2     # cmd1 성공 시 cmd2
cmd1 || cmd2     # cmd1 실패 시 cmd2
cmd1 ; cmd2      # 무조건 둘 다

# guard 패턴
mkdir -p /opt/app && cd /opt/app
[ -f /etc/passwd ] || { echo "missing"; exit 1; }
```

---

## 8. quoting

```bash
echo $A     # split (공백 분리)
echo "$A"   # 한 단어 (★ 거의 항상 quote)
echo '$A'   # 그대로 ($A)

# 가장 흔한 버그
files=$(find . -name "*.log")
for f in $files; do          # ❌ 공백 있는 파일명 깨짐
    rm "$f"
done

# ✅
find . -name "*.log" -print0 | xargs -0 rm
# 또는
while IFS= read -r -d '' f; do
    rm "$f"
done < <(find . -name "*.log" -print0)
```

---

## 9. exit code

```bash
$?                # 직전 명령의 exit code (0 = success)
cmd && echo "ok" || echo "fail"

exit 0            # 성공
exit 1            # 일반 실패
exit 127          # command not found
```

---

## 10. 함정

1. **`$A` quote 안 함** — 공백 / glob expansion.
2. **`for f in $(ls)`** — 파일명에 공백 / newline 깨짐 → `find -print0`.
3. **redirect 순서** `2>&1 > file` ≠ `> file 2>&1`.
4. **set -e 미사용** — script 가 error 무시하고 진행.
5. **`rm -rf $VAR/`** — VAR 비어있으면 `rm -rf /` 재앙.

---

## 11. 관련

- [[linux|↑ linux]]
- [[shell-scripting]]
- [[file-permissions]]
