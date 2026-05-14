---
title: "Networking in RN — HTTP / WebSocket hub"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, networking, axios, websocket, hub]
---

# Networking in RN

**[[../react-native|↑ RN Hub]]**

> HTTP = frontend 와 거의 동일. WebSocket / STOMP / Socket.IO = job-answer-app-rn 의 채팅.

## 1. 가능한 통신 방식

| | 라이브러리 / API | 용도 |
| --- | --- | --- |
| **fetch** | RN 내장 | 간단한 HTTP |
| **axios** | yarn add axios | HTTP + interceptor |
| **WebSocket** | RN 내장 `WebSocket` | 양방향 실시간 |
| **STOMP** | `@stomp/stompjs` | WebSocket 위 메시지 프로토콜 (job-answer-app-rn) |
| **Socket.IO** | `socket.io-client` | 자체 프로토콜 (Node 서버 쪽 많이 씀) |
| **gRPC** | 드물게 | 양방향 streaming |
| **GraphQL** | Apollo / urql | 서버에 따라 |

## 2. HTTP — fetch vs axios

```ts
// fetch (내장)
const r = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ name: '유철' }),
});
if (!r.ok) throw new Error(r.status);
const data = await r.json();

// axios (라이브러리)
const { data } = await axios.post('https://api.example.com/users', { name: '유철' });
```

→ axios 가 인터셉터 + 자동 JSON + 자동 throw. job-answer-app-rn 도 axios 사용 추정.

자세히 [[axios-fetch]].

## 3. WebSocket — 실시간 메시지

```ts
const ws = new WebSocket('wss://chat.example.com/ws');

ws.onopen = () => {
  ws.send(JSON.stringify({ type: 'hello' }));
};
ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  console.log(msg);
};
ws.onerror = (e) => console.error(e);
ws.onclose = () => console.log('closed');

// 종료
ws.close();
```

→ HTTP 의 일회성 요청과 달리 **연결 유지 + 양방향**.

## 4. STOMP — WebSocket 위 메시지 프로토콜

job-answer-app-rn 사용 (`@stomp/stompjs`, `@stomp/rx-stomp`).

```bash
yarn add @stomp/stompjs
```

```ts
import { Client } from '@stomp/stompjs';

const client = new Client({
  brokerURL: 'wss://chat.example.com/ws',
  connectHeaders: { Authorization: `Bearer ${token}` },
  reconnectDelay: 5000,
  onConnect: () => {
    // 구독
    client.subscribe('/topic/room/123', (message) => {
      console.log(message.body);
    });

    // 보내기
    client.publish({
      destination: '/app/room/123/send',
      body: JSON.stringify({ text: '안녕' }),
    });
  },
});

client.activate();
```

→ Spring 의 STOMP 서버와 양방향. 자세히 [[websocket-stomp]].

## 5. Socket.IO

```bash
yarn add socket.io-client
```

```ts
import { io } from 'socket.io-client';

const socket = io('https://server.example.com', {
  auth: { token },
});

socket.on('connect', () => console.log('connected'));
socket.on('message', (msg) => console.log(msg));
socket.emit('send', { text: 'hi' });
```

→ Node 백엔드와 자주 사용.

## 6. 환경 설정

### iOS — App Transport Security (ATS)

`Info.plist`:
```xml
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSAllowsArbitraryLoads</key>
  <true/>           <!-- HTTPS 강제 완화 (dev only) -->
</dict>
```

→ iOS 9+ 부터 HTTPS 만 허용. dev 서버가 http 면 ATS 완화.

### Android — Cleartext Traffic

`AndroidManifest.xml`:
```xml
<application android:usesCleartextTraffic="true">
  ...
</application>
```

→ Android 9+ HTTPS 만 기본. dev http 허용.

→ **production 은 HTTPS**. ATS / cleartext 완화는 release 빌드에서 끄기.

## 7. timeout / 재시도

```ts
// axios
const api = axios.create({ timeout: 10_000 });

// retry — axios-retry
yarn add axios-retry

import axiosRetry from 'axios-retry';
axiosRetry(api, { retries: 3, retryDelay: axiosRetry.exponentialDelay });
```

→ 모바일은 네트워크 불안정. retry + timeout 필수.

## 8. 네트워크 상태 감지

```bash
yarn add @react-native-community/netinfo
```

```tsx
import NetInfo from '@react-native-community/netinfo';
import { useEffect, useState } from 'react';

function NetworkBanner() {
  const [online, setOnline] = useState(true);
  useEffect(() => {
    const unsub = NetInfo.addEventListener((state) => {
      setOnline(!!state.isConnected);
    });
    return unsub;
  }, []);

  if (online) return null;
  return (
    <View style={{ backgroundColor: 'red', padding: 8 }}>
      <Text style={{ color: 'white', textAlign: 'center' }}>오프라인</Text>
    </View>
  );
}
```

### react-query 와 결합 — 자동 refetch on reconnect
```ts
import { onlineManager } from '@tanstack/react-query';

onlineManager.setEventListener((setOnline) => {
  return NetInfo.addEventListener((state) => setOnline(!!state.isConnected));
});
```

## 9. file 업로드 (multipart)

```ts
const fd = new FormData();
fd.append('file', {
  uri: 'file:///path/to/photo.jpg',     // local URI
  type: 'image/jpeg',
  name: 'photo.jpg',
} as any);
fd.append('description', '설명');

await axios.post('/upload', fd, {
  headers: { 'Content-Type': 'multipart/form-data' },
  onUploadProgress: (e) => setProgress(Math.round((e.loaded / e.total!) * 100)),
});
```

→ image-picker / camera 의 결과를 곧장 업로드.

## 10. dev 환경 — localhost 접근

iOS sim: `http://localhost:8080` 그대로 사용 가능.
Android emulator: `localhost` 가 emulator 자기 자신 → **`http://10.0.2.2:8080`** 사용.

실기기: 같은 wifi 의 PC 의 IP 사용 (`http://192.168.0.5:8080`).

## 11. 학습 우선순위

1. **[[axios-fetch]]** — HTTP + interceptor.
2. **react-query 도입** — [[../../../frontend/react/server-state/react-query|frontend 의 react-query]].
3. **[[websocket-stomp]]** — 채팅 / 실시간.
4. **NetInfo + onlineManager** — 오프라인 처리.
5. **file 업로드 + progress**.

## 12. 함정

1. **localhost 안 됨 (Android)** — `10.0.2.2`.
2. **HTTPS 강제** — dev http 면 ATS / cleartext 완화 필요. production 은 HTTPS.
3. **fetch 의 4xx/5xx resolve** — `.ok` 체크 또는 axios.
4. **timeout 짧음** — 모바일 네트워크 불안정. 10초+ 권장.
5. **WebSocket 의 token 만료** — 재연결 시 새 token. STOMP onWebSocketError 처리.
6. **file upload 의 type field** — RN 의 FormData 는 `{uri, type, name}` 객체.
7. **NetInfo cleanup 누락** — 메모리 누수.

## 13. 다음 단계

- [[axios-fetch]] — HTTP
- [[websocket-stomp]] — 실시간

## 14. 관련

- [[../react-native]]
- [[../auth/auth]] — 인증 + token
- [[../../../frontend/react/http/http|frontend 의 http]]
- [[../../../frontend/react/server-state/react-query|react-query]]
