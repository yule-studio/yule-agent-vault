---
title: "HTTP Pipelining — 실패한 시도"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T00:45:00+09:00
tags:
  - network
  - http
  - performance
  - pipelining
---

# HTTP Pipelining — 실패한 시도

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | HoL Blocking / HTTP/2 의 진화 |

**[[performance|↑ Performance]]** · **[[../http|↑↑ HTTP]]**

---

## 1. 한 줄 정의

HTTP/1.1 의 기능 — **응답 안 기다리고 여러 요청 동시 송신**. HoL Blocking 등으로
사장. HTTP/2 의 멀티플렉싱이 대체.

---

## 2. 동작

### 일반 (Pipelining 없음)
```
Req 1 → Resp 1 → Req 2 → Resp 2 → ...
```

### Pipelining
```
Req 1 →
Req 2 →
Req 3 →
       ← Resp 1
       ← Resp 2  ← 순서 그대로
       ← Resp 3
```

→ 이론적으로 RTT 절약.

---

## 3. 한계 — HoL Blocking

```
Req 1 (느림) → Resp 1 (10s 후)
Req 2 (빠름) → Resp 2 (1s 가능하지만 stuck)
Req 3 (빠름) → Resp 3 (stuck)
```

→ **응답은 순서대로** — 첫 응답 늦으면 모든 응답 stuck.

---

## 4. 미들박스 호환성 X

- 일부 Proxy / 방화벽 가 pipelined request 무시 / 잘못 처리
- ISP 의 transparent proxy
- 응답 섞여서 전달
- → 안정성 부족

---

## 5. 브라우저의 기본 비활성

| 브라우저 | Pipelining |
| --- | --- |
| Chrome | 기본 X |
| Firefox | 옛 옵션 — 비활성 (about:config) |
| Safari | 기본 X |
| Opera | 옛 시도 — 사장 |

→ 사실상 사용 X.

---

## 6. HTTP/2 의 해결

```
Multiplexing — 한 TCP 안 수십 stream
각 stream 독립 — 한 stream 의 지연이 다른 영향 X (응용 레벨)
```

자세히 → [[../versions/http-2]]

### Pipelining 과 Multiplexing 차이

| 측면 | HTTP/1.1 Pipelining | HTTP/2 Multiplexing |
| --- | --- | --- |
| 응답 순서 | 순서 강제 (FIFO) | 자유 |
| HoL Blocking | 응용 레벨 | 없음 (응용), TCP 레벨 잔존 |
| 미들박스 호환 | 부족 | 표준 |
| 보편 | 비활성 | 표준 |

---

## 7. HTTP/3 의 완전 해결

QUIC 의 stream 독립성:
- TCP 의 패킷 손실 시 모든 stream stuck → QUIC 은 stream 별 독립
- HoL Blocking 진정 해결

자세히 → [[../versions/http-3-quic]]

---

## 8. 실제 사용 — 거의 X

```
Production 환경:
- HTTP/1.1 → 6 connection (병렬 TCP)
- HTTP/2 → 1 connection (멀티 stream)
- HTTP/3 → 1 QUIC connection (멀티 stream + 0-RTT)
```

→ Pipelining 의 자리 없음.

---

## 9. 일부 특수 환경

- **테스트 / 학술 연구**
- **OkHttp 옛 옵션**
- **모바일 단방향 환경** (옛)

→ 모두 사장. HTTP/2/3 사용.

---

## 10. curl 의 옵션

```bash
# Pipelining (libcurl 옛 옵션)
# 현재는 기본 X — 권장 X
```

---

## 11. 함정

### 함정 1 — Pipelining 이 Keep-Alive 와 같다는 가정
- Keep-Alive = 연결 재사용
- Pipelining = 응답 안 기다리고 요청 송신

### 함정 2 — HTTP/2 의 multiplexing 을 pipelining 이라 부름
- 다른 메커니즘
- 정확한 용어 사용

### 함정 3 — pipelining 활성화 시도
모던 브라우저 / 라이브러리 — 비활성. HTTP/2/3 사용.

---

## 12. 역사적 의의

- HTTP/1.1 의 야망
- 실패가 HTTP/2 의 동기
- "응답 순서 강제" 의 교훈 — stream 독립성

---

## 13. 학습 자료

- **RFC 9112** Section 9.3.2 (Pipelining)
- "HTTP Pipelining: Not So Fast" — historical articles
- HTTP/2 explained

---

## 14. 관련

- [[performance]] — Performance hub
- [[keep-alive]] — 다른 메커니즘
- [[../versions/http-2]] — Multiplexing 의 대체
- [[../versions/http-3-quic]] — HoL 완전 해결
