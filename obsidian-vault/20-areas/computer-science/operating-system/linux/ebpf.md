---
title: "eBPF — Extended Berkeley Packet Filter"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T17:15:00+09:00
tags:
  - operating-system
  - linux
  - ebpf
---

# eBPF

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | eBPF 개념 + 도구 |

**[[linux|↑ Linux hub]]**

---

## 1. 한 줄

응용이 작성한 **kernel 안의 안전한 가상머신 프로그램**. 추적 / 관찰 / 네트워크 / 보안의 새 패러다임.

Linux 3.18+. Brendan Gregg / Alexei Starovoitov 등.

---

## 2. 왜

```
전통적: 응용 ↔ kernel 의 정해진 인터페이스 (syscall, proc)
eBPF:   응용이 kernel 안에 동적 hook → 안전 검증 → 빠른 분석 / 정책
```

장점:
- 매우 빠름 (JIT)
- 안전 (verifier 가 종료 보장)
- 동적 (재부팅 없이)
- 광범위 (kprobe / tracepoint / XDP / cgroup / LSM)

---

## 3. 응용 영역

| 영역 | 도구 / 예 |
| --- | --- |
| **Observability** | bcc-tools, bpftrace, libbpf-tools |
| **Network** | Cilium (CNI), Katran (LB), Calico |
| **Security** | Tetragon, Falco, BPF LSM |
| **Profiling** | parca, pyroscope (옛 native) |
| **Tracing** | execsnoop, opensnoop, tcptop |
| **XDP / TC** | 고성능 packet processing |

---

## 4. Hook 종류

| Hook | 의미 |
| --- | --- |
| **kprobe** | kernel 함수 진입 |
| **uprobe** | userspace 함수 |
| **tracepoint** | 미리 정의된 kernel event |
| **USDT** | userspace static tracepoint |
| **perf_event** | hardware counter |
| **XDP** | NIC driver 입구 (가장 빠른 packet 처리) |
| **TC** | traffic control |
| **socket filter / cgroup hook** | socket / cgroup |
| **LSM** | 보안 정책 |
| **fentry / fexit** | kprobe 진화 (5.5+) |

---

## 5. bpftrace — 빠른 시작

awk 비슷한 DSL. 한 줄로 강력.

```bash
sudo apt install bpftrace

# 가장 자주 호출되는 syscall
sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[probe] = count(); }'

# openat 호출 추적
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s -> %s\n", comm, str(args->filename)); }'

# disk latency 분포
sudo bpftrace -e 'kprobe:blk_account_io_start { @start[arg0] = nsecs; }
                  kprobe:blk_account_io_done {
                    @us = hist((nsecs - @start[arg0]) / 1000);
                    delete(@start[arg0]);
                  }'

# CPU profile
sudo bpftrace -e 'profile:hz:99 { @[comm, kstack] = count(); }'
```

---

## 6. bcc-tools

Python + C 의 한 묶음. 풍부한 ready-to-use scripts.

```bash
sudo apt install bpfcc-tools

# 가장 유명한 도구들 (-bpfcc suffix on Ubuntu)
sudo execsnoop-bpfcc                    # exec 추적
sudo opensnoop-bpfcc                     # open 추적
sudo tcpconnect-bpfcc                    # TCP connect
sudo tcpaccept-bpfcc
sudo biolatency-bpfcc                    # block I/O latency 분포
sudo biosnoop-bpfcc
sudo cachestat-bpfcc                     # page cache hit
sudo runqlat-bpfcc                       # run queue latency
sudo offcputime-bpfcc                    # off-CPU profiling
sudo profile-bpfcc 30                    # CPU profile (perf 대안)
sudo tcptop-bpfcc
sudo killsnoop-bpfcc                     # signal
sudo memleak-bpfcc -p $PID
sudo gethostlatency-bpfcc                # DNS latency
sudo statsnoop-bpfcc
sudo execsnoop-bpfcc -n 'java'
```

`man bcc` 의 카탈로그 확인.

---

## 7. libbpf-tools — 새 표준

bcc 의 Python 의존 무거움 → **libbpf-tools** (C / CO-RE).

```bash
sudo apt install libbpf-tools
sudo execsnoop                          # bpfcc 와 같은 동작
sudo opensnoop
```

production 친화 (작은 binary, 빠른 시작).

---

## 8. eBPF map

```
응용 ↔ kernel program 간 데이터 공유
- BPF_MAP_TYPE_HASH
- BPF_MAP_TYPE_ARRAY
- BPF_MAP_TYPE_RINGBUF
- BPF_MAP_TYPE_PERF_EVENT_ARRAY
- BPF_MAP_TYPE_LRU_HASH
- ...
```

- key-value
- per-CPU (no lock)
- LRU eviction

---

## 9. Verifier

eBPF 프로그램 load 시:
- bounded loop only (5.3+ 부터 bounded loop OK)
- 종료 보장 (1M instruction limit)
- 메모리 안전성 검증
- 포인터 검사

→ kernel 안에서 무한 loop / crash 불가.

---

## 10. JIT

eBPF bytecode → x86 / ARM native code. interpreter 대비 10x+ 빠름.

```bash
sysctl net.core.bpf_jit_enable
```

---

## 11. XDP — eXpress Data Path

```
NIC 의 RX 직후 → eBPF
- packet 처리 / drop / forward
- DPDK 수준 성능 + kernel 친화
- DDoS 방어, load balancer
```

Cloudflare 의 magic-firewall, Facebook 의 Katran.

---

## 12. cilium / CNI

Kubernetes 의 CNI plugin — kube-proxy 대체:
- service load-balancing in eBPF
- NetworkPolicy
- observability (Hubble)

iptables 의 N rule → eBPF 의 hash-lookup → 더 빠름.

---

## 13. Falco / Tetragon — 보안

```yaml
- rule: write to /etc/passwd
  condition: open_write and fd.name = /etc/passwd
  output: ...
  priority: WARNING
```

container / 시스템의 의심 활동 실시간 감지. 모두 eBPF 기반.

---

## 14. 학습 곡선

| 단계 | |
| --- | --- |
| 1 | bpftrace one-liner 부터 |
| 2 | bcc-tools 실행 / 결과 해석 |
| 3 | bpftrace script (.bt) 작성 |
| 4 | libbpf + C (`*.bpf.c`) |
| 5 | XDP / TC / LSM 등 advanced |

---

## 15. 보안 / 권한

```bash
# 옛 — root
# 새 (5.8+) — CAP_BPF
sudo setcap cap_bpf,cap_perfmon=ep ./tool

# kernel 옵션
sysctl kernel.unprivileged_bpf_disabled
```

unprivileged eBPF = 보안 위험 (CVE 가 있었음). 일반적으로 disabled.

---

## 16. eBPF 의 future

- Windows 도 eBPF 지원 (Microsoft)
- BPF LSM — 보안 정책
- BPF Type Format (BTF) — portability
- io_uring 통합
- CO-RE (Compile Once - Run Everywhere)

→ Linux kernel observability / network / security 의 새 표준.

---

## 17. 실전 예

### 17.1 누가 /etc/passwd 를 열었나

```bash
sudo bpftrace -e '
  tracepoint:syscalls:sys_enter_openat
  /str(args->filename) == "/etc/passwd"/ {
    printf("%s (pid %d)\n", comm, pid);
  }
'
```

### 17.2 TCP 연결 추적

```bash
sudo tcpconnect-bpfcc
```

### 17.3 syscall latency 분포

```bash
sudo funclatency-bpfcc 'syscalls:sys_enter_read'
```

### 17.4 CPU profile (low overhead)

```bash
sudo profile-bpfcc -F 99 -ad 30 -p $PID > out.stacks
./flamegraph.pl out.stacks > out.svg
```

---

## 18. 함정

### 18.1 kernel version 의존
일부 도구 = 새 kernel 만. 5.8+ 권장.

### 18.2 BTF 부재
구버전 kernel = BTF 없음 → CO-RE 안 됨.

### 18.3 verifier 의 거부
복잡한 로직이 verifier 의 한계 초과. 단순화.

### 18.4 overhead — uprobe
userspace 함수 trace 는 비쌈. tracepoint > kprobe > uprobe.

### 18.5 lost samples
ringbuf 가득 차면 drop. 처리량 / map 크기.

### 18.6 unprivileged bpf
보안 위험. disabled (기본).

### 18.7 컨테이너의 host kernel
container 안 도구가 host kernel 의 trace — host 권한.

---

## 19. 학습 자료

- **BPF Performance Tools** — Brendan Gregg
- **Learning eBPF** — Liz Rice
- **ebpf.io** — Cilium / Foundation
- **bcc / bpftrace / libbpf-tools** github
- **Brendan Gregg blog**

---

## 20. 관련

- [[strace-perf]] — perf
- [[monitoring]]
- [[../syscall/syscall]]
- [[../../network/network]]
- [[linux]] — Linux hub
