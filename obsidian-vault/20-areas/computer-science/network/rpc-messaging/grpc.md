---
title: "gRPC — Modern RPC"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T08:05:00+09:00
tags:
  - network
  - rpc
  - grpc
  - protobuf
---

# gRPC — Modern RPC

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | HTTP/2 / Protobuf / Streaming |

**[[rpc-messaging|↑ RPC/Messaging]]** · **[[../network|↑↑ network hub]]**

---

## 메타 표

| 항목 | 값 |
| --- | --- |
| **시작** | Google (2015), CNCF |
| **전송** | HTTP/2 (HTTP/3 베타) |
| **IDL** | Protocol Buffers (proto3) |
| **인코딩** | Binary (Protobuf) |
| **포트** | 보통 50051 |

---

## 1. 한 줄 정의

**HTTP/2 + Protobuf** 기반 RPC. Google 의 내부 Stubby → 오픈 (2015). 마이크로서비스 / streaming 의 사실상 표준.

---

## 2. 왜 gRPC?

### REST 의 한계
- JSON 무거움 (parsing 비용)
- HTTP/1.1 의 한 connection 한 요청
- Streaming 어려움 (long poll, SSE)
- Schema 약함 (OpenAPI 선택)
- Polyglot — 클라이언트 마다 SDK 수동

### gRPC 해결
- Protobuf — 작고 빠름
- HTTP/2 — multiplexing
- Streaming native
- Schema 강제 (.proto)
- 자동 codegen — Go / Java / Python / Node.js / Ruby / C++ / ...

---

## 3. .proto — IDL

```protobuf
syntax = "proto3";

package user;

service UserService {
    rpc GetUser (GetUserRequest) returns (User);
    rpc ListUsers (ListUsersRequest) returns (stream User);
    rpc CreateUsers (stream CreateUserRequest) returns (CreateUserResponse);
    rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
    repeated string tags = 4;
}

message GetUserRequest {
    int64 id = 1;
}

message ListUsersRequest {
    int32 page_size = 1;
    string page_token = 2;
}
```

### Field number
- 1-15 — 1 byte
- 16-2047 — 2 byte
- 영구 (변경 X)
- Deprecated — `reserved`

### Codegen
```bash
protoc --go_out=. --go-grpc_out=. user.proto
protoc --python_out=. --grpc_python_out=. user.proto
```

---

## 4. 4 가지 통신 모드

### Unary
```protobuf
rpc GetUser (GetUserRequest) returns (User);
```

→ 일반 RPC.

### Server Streaming
```protobuf
rpc ListUsers (Request) returns (stream User);
```

→ 서버가 여러 응답.

### Client Streaming
```protobuf
rpc Upload (stream Chunk) returns (UploadStatus);
```

→ 클라이언트가 여러 요청 → 서버 한 응답.

### Bidirectional Streaming
```protobuf
rpc Chat (stream Message) returns (stream Message);
```

→ 양방향 동시.

---

## 5. HTTP/2 위의 매핑

### 요청
```
HEADERS:
:method = POST
:scheme = https
:path = /user.UserService/GetUser
content-type = application/grpc
grpc-encoding = gzip (선택)
te = trailers

DATA: <Protobuf message>
```

### 응답
```
HEADERS:
:status = 200
content-type = application/grpc

DATA: <Protobuf response>

TRAILERS:
grpc-status = 0  (OK)
grpc-message = (optional)
```

### 한 stream — 한 RPC
- HTTP/2 multiplexing — 한 connection 의 여러 RPC 동시

---

## 6. 상태 / 에러 코드

| Code | 이름 | 설명 |
| --- | --- | --- |
| 0 | OK | 성공 |
| 1 | CANCELLED | 클라가 취소 |
| 2 | UNKNOWN | 알 수 없음 |
| 3 | INVALID_ARGUMENT | 잘못된 인자 |
| 4 | DEADLINE_EXCEEDED | 타임아웃 |
| 5 | NOT_FOUND | 없음 |
| 6 | ALREADY_EXISTS | 중복 |
| 7 | PERMISSION_DENIED | 권한 |
| 8 | RESOURCE_EXHAUSTED | 자원 |
| 9 | FAILED_PRECONDITION | 상태 |
| 10 | ABORTED | 취소 (재시도 OK) |
| 11 | OUT_OF_RANGE | 범위 |
| 12 | UNIMPLEMENTED | 미구현 |
| 13 | INTERNAL | 내부 |
| 14 | UNAVAILABLE | 사용 불가 (재시도) |
| 15 | DATA_LOSS | 데이터 손실 |
| 16 | UNAUTHENTICATED | 인증 |

→ HTTP status 보다 풍부.

---

## 7. Deadline / Timeout

```go
ctx, cancel := context.WithTimeout(context.Background(), 100*time.Millisecond)
defer cancel()

user, err := client.GetUser(ctx, &pb.GetUserRequest{Id: 1})
```

### 전파
- 클라이언트가 server 로 timeout 헤더 전달
- 중간 서비스도 같은 deadline
- 자동 전파 (context)

→ Distributed deadline 의 핵심.

---

## 8. 인증 / 보안

### TLS
- 거의 모든 production — TLS
- 자체 서명 / Let's Encrypt / 사설 CA

### mTLS
- Service Mesh — 모든 서비스 cert
- Istio / Linkerd 자동

### Token (JWT / API key)
- Metadata 헤더
- Interceptor 검증

자세히 → [[../tls-ssl/mtls]]

---

## 9. Interceptor / Middleware

### Server interceptor (Go)
```go
func authInterceptor(ctx context.Context, req interface{}, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (interface{}, error) {
    md, _ := metadata.FromIncomingContext(ctx)
    token := md.Get("authorization")
    if !valid(token) {
        return nil, status.Error(codes.Unauthenticated, "invalid token")
    }
    return handler(ctx, req)
}

server := grpc.NewServer(grpc.UnaryInterceptor(authInterceptor))
```

### 사용
- Auth / Logging / Tracing / Metrics
- OpenTelemetry / Prometheus
- Rate limit / circuit breaker

---

## 10. gRPC-Web — 브라우저

### 문제
- 브라우저 — HTTP/2 의 trailers 접근 X
- gRPC raw 사용 불가

### gRPC-Web
- 별도 spec — HTTP/1.1 + base64 encoding
- Proxy (Envoy) 가 변환

### 흐름
```
Browser (gRPC-Web) → Envoy (gRPC-Web filter) → Backend (gRPC)
```

### 한계
- Streaming — server streaming 만 (bi-directional 불가)

### Connect-Web (Buf)
- gRPC-Web 의 모던 대체
- Connect protocol — 단순 / 표준

---

## 11. Load Balancing — gRPC

### 문제
- HTTP/2 — 한 connection 의 multiplexing → 한 LB pin
- 옛 L4 LB — RPC 분산 X (한 backend 만)

### 해결
- **L7 LB (ALB / Envoy)** — RPC 단위 분산
- **Client-side LB** — gRPC 클라이언트가 여러 backend 직접
- **xDS** — Envoy 의 dynamic config

### Server reflection
- 클라이언트가 server 의 .proto 동적 조회
- grpcurl 같은 도구

---

## 12. 비교 — REST vs gRPC

| | REST | gRPC |
| --- | --- | --- |
| **포맷** | JSON / XML | Protobuf |
| **크기** | 큼 | 작음 (1/3-1/10) |
| **속도** | 느림 | 빠름 |
| **HTTP** | 1.1+ | 2 (3 베타) |
| **Streaming** | (SSE / WebSocket) | native |
| **Schema** | OpenAPI (선택) | proto (필수) |
| **Browser** | 좋음 | gRPC-Web 필요 |
| **디버깅** | curl / browser | grpcurl 필요 |
| **버전** | URL / 헤더 | field number |
| **사용** | Public API | 내부 microservice |

---

## 13. 도구

### grpcurl — curl 의 gRPC 버전
```bash
grpcurl -plaintext localhost:50051 user.UserService/GetUser -d '{"id": 1}'
grpcurl -plaintext localhost:50051 list
grpcurl -plaintext localhost:50051 describe user.UserService
```

### BloomRPC / Postman / Insomnia
- GUI 도구

### Buf
- modern proto 도구 — linting / breaking change detect / registry

### protoc-gen-validate
- proto 의 validation rules

---

## 14. Connect — Buf 의 새 RPC

### 정의
- gRPC + REST 호환
- 같은 .proto → gRPC, gRPC-Web, Connect (HTTP)

### 특징
- Connect 는 HTTP/1.1 / 1.0 호환
- 브라우저 native
- gRPC client 도 호출 가능

### 점진 도입
- gRPC 대체 (호환)

---

## 15. 함정

### 함정 1 — Proto field number 변경
영구. 변경 시 — 데이터 호환 깨짐.

### 함정 2 — Required → optional (proto3)
proto3 — 모든 field optional. 강제 검증은 application 책임.

### 함정 3 — Long-lived stream 의 LB
한 backend pin. Stream 단위 LB / drain.

### 함정 4 — Deadline 전파 안 함
중간 서비스 — 자체 timeout 만. context propagation 필수.

### 함정 5 — HTTP/2 의 HoL blocking (옛)
TCP HoL. HTTP/3 (QUIC) 가 해결.

### 함정 6 — gRPC-Web 의 bi-directional 미지원
브라우저 — Server streaming 만. Connect-Web / WebSocket fallback.

### 함정 7 — Browser dev tool 미지원
Network 탭에서 binary 만 보임. proto descriptor 필요.

---

## 16. 학습 자료

- gRPC 공식 docs
- "gRPC: Up and Running" (O'Reilly)
- Buf docs (Connect)
- "Service Mesh" Istio / Linkerd docs

---

## 17. 관련

- [[rpc-messaging]] — Hub
- [[graphql]] — 비교
- [[../http/versions/http-2]] — gRPC 기반
- [[../tls-ssl/mtls]] — Service-to-service
