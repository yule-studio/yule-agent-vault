---
title: "Test scenarios — AC 매핑"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:12:00+09:00
tags: [backend, java-spring, api-design, chat, testing]
---

# Test scenarios — AC 매핑

**[[testing|↑ hub]]**

---

| AC | 테스트 | 도구 |
| --- | --- | --- |
| R-1 DIRECT 생성 | RoomServiceIT.directPairUnique | TC |
| R-2 그룹 생성 | RoomServiceIT.groupOwner | TC |
| R-5 max member | RoomServiceIT.maxMemberReject | TC |
| M-1 TEXT send | MessageStompIT.sendReceive | WebSocketStompClient |
| M-7 XSS | XssSanitizerTest | Unit |
| M-9 차단 send X | BlockFilterIT | TC |
| M-10 rate limit | RateLimiterIT.10per1s | TC |
| RD-1 1:1 읽음 | ReadReceiptIT.directRead | TC |
| RD-2 그룹 안 읽은 수 | ReadReceiptIT.groupUnreadCount | TC |
| MD-1 multi session | MultiDeviceIT.twoSessionsSameUser | WebSocketStompClient |
| MD-2 다른 device sync | MultiDeviceIT.crossDevice | TC + Redis |
| MD-5 cross-node | CrossNodeIT.nodeAtoB | TC 2 nodes |
| P-1 presence | PresenceIT.onlineHeartbeat | TC + Redis |
| PUSH-1 offline push | PushFallbackIT.offlineCallsNotif | mocked notification |
| A-1 사진 send | AttachmentIT.imageThumbnail | LocalStack S3 |
| S-1 JWT auth | StompAuthIT.invalidJwt401 | WebSocketStompClient |
| S-3 rate | (위 M-10 동일) | |
| S-7 active conn | LoadIT.10kConnections | k6 |

---

## 관련

- [[testing|↑ hub]]
- [[../requirements]]
