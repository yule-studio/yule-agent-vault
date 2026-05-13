---
title: "Zero-Copy — sendfile / splice / mmap"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T13:30:00+09:00
tags:
  - operating-system
  - io
  - zero-copy
---

# Zero-Copy

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | sendfile / splice / DMA |

**[[io|↑ I/O hub]]**

---

## 1. 한 줄

데이터를 user ↔ kernel 사이에서 **불필요한 복사 없이** 디스크 → 네트워크 / 다른 곳으로 전달.

---

## 2. 일반 read + write

```c
char buf[4096];
while ((n = read(file_fd, buf, 4096)) > 0)
    write(socket_fd, buf, n);
```

복사 흐름:
```
1. DMA: 디스크 → kernel page cache
2. copy: kernel page cache → user buf (CPU)
3. copy: user buf → kernel socket buffer (CPU)
4. DMA: kernel socket buffer → NIC
```

→ **4 번 복사** (DMA 2, CPU 2). 컨텍스트 스위치 2 회.

---

## 3. sendfile (Linux)

```c
sendfile(out_fd, in_fd, &offset, count);
```

```
1. DMA: 디스크 → kernel page cache
2. (CPU 1번 또는 0번): page cache → socket buffer
3. DMA: socket buffer → NIC
```

→ 2-3 번 복사. 컨텍스트 스위치 1 회.

### 3.1 진짜 zero-copy — Scatter-Gather DMA

NIC 가 page cache 페이지를 직접 DMA → 정말 1 copy (DMA만).

```bash
ethtool -k eth0 | grep gather
```

`sendfile` + 호환 NIC + `MSG_ZEROCOPY` → 진짜 zero-copy.

---

## 4. 사용 사례

- **Nginx** — `sendfile on;` (정적 파일 서빙)
- **Kafka** — 큰 파일 → 네트워크 sendfile
- **Apache** — `EnableSendfile`
- **HAProxy / Envoy** — splice
- **NFS / SMB** — server-side

---

## 5. splice

```c
splice(in_fd, NULL, out_fd, NULL, count, flags);
```

- pipe 를 buffer 로 사용
- 양쪽 중 하나는 pipe 여야
- sendfile 의 일반화

```c
int pipefd[2]; pipe(pipefd);
splice(disk_fd, NULL, pipefd[1], NULL, n, 0);
splice(pipefd[0], NULL, socket_fd, NULL, n, 0);
```

---

## 6. tee

```c
tee(fd_in, fd_out, count, flags);
```

같은 데이터를 두 곳으로 (둘 다 pipe).

---

## 7. mmap + write

```c
char *p = mmap(NULL, sz, PROT_READ, MAP_PRIVATE, file_fd, 0);
write(socket_fd, p, sz);
```

```
1. DMA: 디스크 → kernel page cache
2. (no copy — mmap 으로 매핑)
3. copy: kernel page cache → socket buffer (CPU)
4. DMA: socket buffer → NIC
```

→ 3 복사. user 가 데이터 처리 가능 (압축 / 변형 등).

자세히 → [[../memory/mmap]]

---

## 8. MSG_ZEROCOPY (4.14+)

```c
setsockopt(sock, SOL_SOCKET, SO_ZEROCOPY, &one, sizeof(one));
send(sock, buf, sz, MSG_ZEROCOPY);
```

- 큰 buffer 의 zero-copy send
- 완료 알림 비동기 (errqueue)
- 응용이 buffer lifecycle 관리

---

## 9. io_uring + 등록 buffer

io_uring 에 buffer 등록 시 매 op 의 copy 회피.

```c
io_uring_register_buffers(&ring, iovs, n);
io_uring_prep_read_fixed(sqe, fd, buf, sz, offset, idx);
```

자세히 → [[io-uring]]

---

## 10. RDMA / DPDK / kernel bypass

극단의 zero-copy:
- **RDMA** — NIC ↔ 응용 메모리 직접 (HPC, AI cluster)
- **DPDK** — userspace 의 NIC 드라이버
- **AF_XDP** — eBPF 기반 사용자 공간 packet

→ kernel 우회 = 더 빠름 + 매우 복잡.

---

## 11. 측정

```bash
# strace 로 syscall 흐름
strace -c -e read,write,sendfile ./app

# nginx
# access_log 의 응답 시간 + ethtool -S 의 ZeroCopy stat
```

대용량 파일 서빙은 sendfile 효과 ↑.

---

## 12. 함정

### 12.1 sendfile + 응용 처리
변형 / 압축 필요 시 sendfile X. mmap+write 또는 splice + pipe.

### 12.2 작은 파일
sendfile 의 오버헤드 > 이점. 단순 read+write.

### 12.3 sendfile 의 size 제한
2 GB 기본 (off_t). 큰 파일 = loop.

### 12.4 NIC 호환성
Scatter-Gather DMA 미지원 NIC = CPU copy 발생.

### 12.5 MSG_ZEROCOPY 의 비용
작은 size = 알림 비용 > 이득.

### 12.6 mmap + write 의 page fault
큰 파일 = lazy fault → latency spike. MAP_POPULATE.

### 12.7 TLS / encryption
sendfile + TLS 충돌 — kTLS (kernel TLS) 가 해결 (4.13+).

---

## 13. 학습 자료

- **Brendan Gregg — Zero-Copy**
- **kernel.org** `Documentation/networking/msg_zerocopy.rst`
- **kTLS** docs
- **Kafka — Why Kafka is fast** (sendfile 활용)

---

## 14. 관련

- [[io-models]]
- [[../memory/mmap]]
- [[io-uring]]
- [[io]] — I/O hub
