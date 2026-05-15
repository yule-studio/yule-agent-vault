---
title: "Linux — 함정 모음"
kind: knowledge
project: devops
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T07:02:00+09:00
tags: [devops, linux, pitfalls]
---

# Linux — 함정 모음

**[[linux|↑ linux]]**

---

## 1. file / permission

1. **`chmod 777`** — 모두 쓰기. 보안 재앙.
2. **`~/.ssh/id_rsa` 권한 644** — ssh client 가 거부.
3. **`/tmp` sticky bit 사라짐** — 다른 사용자가 파일 삭제 가능.
4. **`rm -rf $VAR/`** — VAR 비어있으면 `rm -rf /`.
5. **`find -exec rm`** — 공백 파일명 깨짐. `-print0 | xargs -0`.

---

## 2. shell

1. **`set -e` 없음** — error 무시.
2. **`set -u` 없음** — typo 변수 빈 문자열.
3. **`"$VAR"` quote 안 함** — 공백 / glob.
4. **`for f in $(ls)`** — 파일명 공백 깨짐.
5. **redirect 순서** `2>&1 > file` ≠ `> file 2>&1`.
6. **shebang 없음** — `sh` 로 실행 시 bash syntax error.

---

## 3. process / signal

1. **SIGKILL 남발** — cleanup 안 됨 (DB 트랜잭션, file flush).
2. **container PID 1 가 signal 무시** — `tini` / `dumb-init`.
3. **`ulimit -n 1024`** — connection 많을 때 부족.
4. **zombie 누적** — wait() 안 하는 부모.
5. **`kill -9` 으로 config reload** — nginx 는 SIGHUP.
6. **nohup 없이 long task** — terminal 끊김 시 죽음.

---

## 4. systemd

1. **`daemon-reload` 안 함** — unit 변경 후 적용 안 됨.
2. **Type=forking + main PID 추적 X** — 잘못된 PID.
3. **Restart=always + crash loop** — 무한 재시작.
4. **StandardOutput=null** — log 사라짐.
5. **dependency 순환** — boot 행.
6. **`enable` 만 — `start` 별도** — boot 후에야 시작.

---

## 5. network

1. **ping 막힘 (ICMP block)** — nc / curl 로 검증.
2. **DNS cache** — resolvectl flush.
3. **iptables 순서** — 위에서 매칭. DROP 먼저면 ACCEPT X.
4. **conntrack overflow** — `net.netfilter.nf_conntrack_max`.
5. **/etc/hosts override** — 디버깅 시간 낭비.
6. **TIME_WAIT 누적** — ephemeral port 고갈.

---

## 6. ssh

1. **password auth 켜져있음** — brute-force 표적.
2. **`chmod 644 ~/.ssh/id_rsa`** — client 거부.
3. **ForwardAgent 항상** — agent hijack.
4. **bastion 없이 공인 IP** — 공격면 ↑.
5. **공유 key** — audit 불가.
6. **MFA 없음** — phishing 위험.

---

## 7. user / sudo

1. **`usermod -G`** (without `-a`) — 기존 group 사라짐.
2. **`/etc/sudoers` 직접 편집** — syntax error 시 lock-out.
3. **NOPASSWD ALL** — 키 탈취 시 즉시 root.
4. **root 로 daemon 실행** — exploit 시 RCE.
5. **공유 계정** — audit 불가.
6. **systemd User 미지정** — root 로 실행.

---

## 8. package

1. **`apt upgrade` 만, update 안 함** — 오래된 버전.
2. **`curl | sh`** — 무서명 script.
3. **GPG key 검증 안 함** — 위조 repo.
4. **auto major upgrade** — breaking change.
5. **disk full (cache)** — `/var/cache/apt`.

---

## 9. performance

1. **`top` 만 보고 판단** — load/wa/pidstat 종합.
2. **`free` 의 used 만** — `available` 이 진짜 여유.
3. **swap = 무조건 나쁨** 아님 — `si/so` 증가가 진짜.
4. **iostat 첫 줄 사용** — boot 이후 평균. 둘째 줄부터.
5. **load avg = CPU 사용률** 오해 — disk wait 포함.

---

## 10. 일반

1. **time zone 안 맞음** — log 분석 시 혼란. `timedatectl set-timezone UTC` (또는 Asia/Seoul).
2. **NTP 안 함** — clock drift → JWT / TLS / 분산 문제.
3. **disk full** — 서비스 down. 80% alert.
4. **inode 부족** — disk 공간은 남는데 파일 생성 안 됨.
5. **journal log 무한 누적** — `SystemMaxUse` 설정.
6. **shell history 신뢰** — `.bash_history` 가 모든 사용자에 같지 X.

---

## 11. 관련

- [[linux|↑ linux]]
