---
title: "RPC / Messaging — Hub"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T08:00:00+09:00
tags:
  - network
  - rpc
  - messaging
  - grpc
---

# RPC / Messaging — Hub

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | gRPC / GraphQL / WebSocket / MQTT / AMQP / WebRTC |

**[[../network|↑ network hub]]** · **[[../osi-7-layer/osi-7-layer|↑↑ OSI]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **계층** | L7 |
| **목적** | 서비스 간 통신 / 메시징 |
| **양식** | RPC (요청-응답) / Pub-Sub / Streaming |

---

## 1. 한 줄 정의

**서비스 간 통신 프로토콜** — REST 의 한계를 극복하는 다양한 방식 (효율 / 양방향 / 실시간 / 메시지 큐).

---

## 2. 프로토콜 — 한눈

| 프로토콜 | 양식 | 전송 | 사용 |
| --- | --- | --- | --- |
| **REST** | 요청-응답 | HTTP | 일반 API |
| **gRPC** | RPC / streaming | HTTP/2 | 마이크로서비스 |
| **GraphQL** | 쿼리 | HTTP / WS | 클라이언트 친화 |
| **WebSocket** | 양방향 | TCP (Upgrade) | 실시간 |
| **SSE** | 단방향 push | HTTP | 알림 |
| **MQTT** | Pub-Sub | TCP | IoT |
| **AMQP** | 큐 | TCP | 엔터프라이즈 |
| **Kafka** | Log / stream | TCP | 빅데이터 |
| **WebRTC** | P2P | UDP | 화상 |

---

## 3. RPC 의 역사

```
RPC (1970s, Sun, ONC RPC)
  ↓
CORBA (1990s) — 복잡
  ↓
SOAP / XML-RPC (1998-2010s) — XML 무거움
  ↓
REST (2000-) — 단순
  ↓
gRPC (2015-) — 효율 / streaming
GraphQL (2015-) — 클라이언트 친화
```

자세히 → [[grpc]], [[graphql]]

---

## 4. 동기 vs 비동기

### 동기 (Request-Response)
- REST / gRPC / GraphQL
- 클라이언트가 응답 기다림
- Coupling 강함

### 비동기 (Messaging)
- MQTT / AMQP / Kafka
- Fire-and-forget 또는 ack
- Decoupling

---

## 5. 통신 패턴

### 5.1 Request-Response
```
Client → Server: request
Client ← Server: response
```

### 5.2 Pub-Sub
```
Publisher → Topic
Subscriber 1 ← Topic
Subscriber 2 ← Topic
```

### 5.3 Streaming
```
Server → Client: chunk 1
Server → Client: chunk 2
... (시간에 걸쳐)
```

### 5.4 Bi-directional
```
Client ↔ Server: 양방향 동시
```

### 5.5 Queue / Worker
```
Producer → Queue
Worker 1 ← Queue (한 메시지 한 worker)
Worker 2 ← Queue
```

---

## 6. 세부 노트

| 노트 | 영역 |
| --- | --- |
| [[grpc]] | gRPC |
| [[graphql]] | GraphQL |
| [[websocket]] | WebSocket |
| [[mqtt-amqp-kafka]] | 메시지 큐 / 스트림 |
| [[webrtc]] | WebRTC |
| [[sse]] | Server-Sent Events |

---

## 7. REST vs gRPC vs GraphQL

| | REST | gRPC | GraphQL |
| --- | --- | --- | --- |
| **포맷** | JSON | Protobuf (binary) | JSON |
| **전송** | HTTP/1.1+ | HTTP/2 | HTTP/1.1+ |
| **스키마** | OpenAPI (선택) | Proto (필수) | SDL (필수) |
| **Streaming** | (옛: long poll), SSE | native | subscription |
| **브라우저** | 좋음 | gRPC-Web 필요 | 좋음 |
| **N+1 / over-fetch** | 가능 | 한 호출 | 한 쿼리 |
| **버전 관리** | URL / 헤더 | proto field number | schema deprecation |
| **사용** | Public API | 내부 마이크로서비스 | 클라이언트 친화 API |

---

## 8. 메시지 큐 — 비교

자세히 → [[mqtt-amqp-kafka]]

| | MQTT | AMQP | Kafka |
| --- | --- | --- | --- |
| **용도** | IoT | Enterprise | Big data / log |
| **모델** | Pub-Sub | Queue + Exchange | Log |
| **부하** | 가벼움 | 보통 | 매우 큼 |
| **순서** | per topic | per queue | per partition |
| **저장** | 짧음 | broker | 장기 (TB+) |
| **표준** | OASIS | ISO/IEC 19464 | de facto |

---

## 9. 실시간 — WebSocket vs SSE vs Long Polling

자세히 → [[websocket]], [[sse]]

| | WebSocket | SSE | Long Polling |
| --- | --- | --- | --- |
| **방향** | 양방향 | 서버 → 클라 | 옛 — 클라 → 서버 |
| **포트** | HTTP | HTTP | HTTP |
| **재연결** | 수동 | 자동 (event-id) | 자동 |
| **브라우저** | 모두 | 모두 (IE X) | 모두 |
| **부하** | 낮음 | 낮음 | 보통 |
| **복잡** | 보통 | 단순 | 단순 |

---

## 10. P2P — WebRTC

자세히 → [[webrtc]]

### 정의
- Peer-to-Peer 브라우저 통신
- 화상 / 음성 / 데이터 채널
- NAT traversal — STUN / TURN / ICE

---

## 11. 학습 자료

- "Designing Data-Intensive Applications" (Kleppmann) — 메시징 챕터
- gRPC / GraphQL / Kafka 공식 docs
- "Enterprise Integration Patterns" (Hohpe)

---

## 12. 관련

- [[../http/methods/methods]] — REST
- [[../osi-7-layer/layer-7-application/layer-7-application]] — L7
- [[../load-balancing/l7-load-balancing]] — gRPC 라우팅
