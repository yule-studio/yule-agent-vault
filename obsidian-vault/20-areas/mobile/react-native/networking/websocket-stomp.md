---
title: "WebSocket / STOMP / Socket.IO — 실시간 채팅"
kind: knowledge
project: mobile
agent: engineering-agent/tech-lead
status: current
tags: [mobile, react-native, websocket, stomp, socket-io, realtime]
---

# WebSocket / STOMP / Socket.IO

**[[networking|↑ Networking Hub]]**

> 양방향 통신. job-answer-app-rn 의 채팅. **3 가지 프로토콜 비교 + 실전 코드**.

## 1. 3 가지 선택지

| | 라이브러리 | 백엔드 | 사용 |
| --- | --- | --- | --- |
| **WebSocket** (raw) | 내장 | 자유 | 단순 |
| **STOMP** | `@stomp/stompjs` | Spring + Stomp Broker | job-answer-app-rn |
| **Socket.IO** | `socket.io-client` | Node + socket.io 서버 | 다른 프로젝트 |

→ **백엔드가 어떤 프로토콜 쓰는지** 가 선택 기준.

## 2. WebSocket — 내장 (가장 단순)

```ts
const ws = new WebSocket('wss://chat.example.com/ws', null, {
  headers: { Authorization: `Bearer ${token}` },
});

ws.onopen = () => {
  console.log('connected');
  ws.send(JSON.stringify({ type: 'hello' }));
};

ws.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  console.log('received', msg);
};

ws.onerror = (e) => console.error(e);

ws.onclose = (e) => console.log('closed', e.code, e.reason);

// 보내기
ws.send(JSON.stringify({ type: 'message', text: 'hi' }));

// 종료
ws.close();
```

### state
- `ws.readyState` — `WebSocket.CONNECTING` (0) / `OPEN` (1) / `CLOSING` (2) / `CLOSED` (3).
- `OPEN` 전 send 하면 throw.

## 3. STOMP — job-answer-app-rn 의 표준

```bash
yarn add @stomp/stompjs text-encoding
```

→ RN 에선 `text-encoding` polyfill 필요 (TextEncoder).

```ts
import 'text-encoding';
import { Client, Frame, Message } from '@stomp/stompjs';

const client = new Client({
  brokerURL: 'wss://chat.example.com/ws',
  connectHeaders: {
    Authorization: `Bearer ${token}`,
  },
  debug: __DEV__ ? console.log : undefined,
  reconnectDelay: 5000,
  heartbeatIncoming: 10000,
  heartbeatOutgoing: 10000,

  onConnect: (frame: Frame) => {
    console.log('connected', frame);

    // 구독
    client.subscribe('/topic/room/123', (message: Message) => {
      const data = JSON.parse(message.body);
      onReceive(data);
    });

    // 보내기
    client.publish({
      destination: '/app/room/123/send',
      body: JSON.stringify({ text: '안녕' }),
    });
  },

  onStompError: (frame) => {
    console.error('STOMP error', frame.headers, frame.body);
  },

  onWebSocketError: (e) => {
    console.error('WebSocket error', e);
  },

  onDisconnect: () => console.log('disconnected'),
});

client.activate();
// 종료
client.deactivate();
```

### custom hook — useStompClient

job-answer-app-rn 의 `src/hooks/useChatSocket.ts` 와 유사:

```ts
// hooks/useChatSocket.ts
import { useEffect, useRef, useState } from 'react';
import { Client } from '@stomp/stompjs';
import AsyncStorage from '@react-native-async-storage/async-storage';

export function useChatSocket(roomId: number) {
  const [messages, setMessages] = useState<Message[]>([]);
  const clientRef = useRef<Client | null>(null);

  useEffect(() => {
    let client: Client | null = null;

    (async () => {
      const token = await AsyncStorage.getItem('access_token');
      if (!token) return;

      client = new Client({
        brokerURL: 'wss://chat.example.com/ws',
        connectHeaders: { Authorization: `Bearer ${token}` },
        reconnectDelay: 5000,
        onConnect: () => {
          client?.subscribe(`/topic/room/${roomId}`, (msg) => {
            setMessages(prev => [...prev, JSON.parse(msg.body)]);
          });
        },
      });

      client.activate();
      clientRef.current = client;
    })();

    return () => {
      client?.deactivate();
    };
  }, [roomId]);

  const send = (text: string) => {
    clientRef.current?.publish({
      destination: `/app/room/${roomId}/send`,
      body: JSON.stringify({ text }),
    });
  };

  return { messages, send };
}
```

### 사용
```tsx
function ChatRoom({ route }) {
  const { roomId } = route.params;
  const { messages, send } = useChatSocket(roomId);
  const [input, setInput] = useState('');

  return (
    <>
      <FlatList data={messages} keyExtractor={m => m.id} renderItem={({ item }) => <Text>{item.text}</Text>} />
      <TextInput value={input} onChangeText={setInput} />
      <Pressable onPress={() => { send(input); setInput(''); }}>
        <Text>보내기</Text>
      </Pressable>
    </>
  );
}
```

## 4. rx-stomp — RxJS 통합

```bash
yarn add @stomp/rx-stomp rxjs
```

```ts
import { RxStomp } from '@stomp/rx-stomp';

const rxStomp = new RxStomp();
rxStomp.configure({
  brokerURL: 'wss://chat.example.com/ws',
  reconnectDelay: 5000,
});
rxStomp.activate();

// 구독 = Observable
const sub = rxStomp.watch('/topic/room/123').subscribe((msg) => {
  console.log(JSON.parse(msg.body));
});

// 종료
sub.unsubscribe();
rxStomp.deactivate();
```

→ RxJS 좋아하면 더 깔끔 (filter / map / debounce).

## 5. Socket.IO

```bash
yarn add socket.io-client
```

```ts
import { io, Socket } from 'socket.io-client';

const socket: Socket = io('https://chat.example.com', {
  auth: { token },
  reconnection: true,
  reconnectionDelay: 1000,
});

socket.on('connect', () => console.log(socket.id));
socket.on('message', (msg) => console.log(msg));
socket.on('disconnect', () => console.log('disconnected'));

socket.emit('joinRoom', { roomId: 123 });
socket.emit('send', { roomId: 123, text: '안녕' });

socket.disconnect();
```

→ STOMP 와 구문 다르지만 사상 동일. Node 백엔드 + socket.io 서버에서.

## 6. 인증 (token 만료 시)

```ts
client.onDisconnect = async () => {
  const newToken = await refreshAccessToken();
  client.connectHeaders = { Authorization: `Bearer ${newToken}` };
  client.activate();    // 새 token 으로 재연결
};
```

→ STOMP / Socket.IO 둘 다 token expire 시 reconnect 처리.

## 7. 백그라운드 / 앱 종료

```tsx
import { AppState } from 'react-native';

useEffect(() => {
  const sub = AppState.addEventListener('change', (status) => {
    if (status === 'background') {
      client.deactivate();     // 절전
    } else if (status === 'active') {
      client.activate();
    }
  });
  return () => sub.remove();
}, []);
```

→ background 에서 long-lived connection 유지하면 OS 가 강제 종료. 의도면 OK.

→ 백그라운드에서도 메시지 받으려면 → **FCM push** ([[../native-features/push-fcm]]).

## 8. 재연결 / heartbeat

```ts
new Client({
  reconnectDelay: 5000,                 // 5초마다 재시도
  heartbeatIncoming: 10000,             // 서버 → client (10초 heartbeat 안 오면 끊김)
  heartbeatOutgoing: 10000,             // client → 서버
});
```

→ heartbeat 으로 dead connection 감지.

## 9. 디버깅

```ts
new Client({
  debug: (msg) => console.log('[STOMP]', msg),     // 모든 frame log
});
```

```bash
# WebSocket 의 traffic 확인 — Chrome DevTools (debugger)
# 또는 Charles / Proxyman 같은 proxy
```

## 10. 함정

1. **`text-encoding` 폴리필 누락** — Stomp 가 TextEncoder 사용. RN 에 없음.
2. **token 만료 → 재연결** — 자동 재연결되지만 같은 옛 token. onDisconnect 에서 refresh.
3. **AppState 무처리** — background 에서 연결 유지 → OS 강제 종료 + 배터리 소모.
4. **subscribe ID 누락** — unsubscribe 못함. 변수에 보관.
5. **`brokerURL` 의 wss / ws** — production HTTPS = wss. dev http = ws.
6. **클라이언트 종료 시 cleanup** — `deactivate()`. unmount 시 unsubscribe.
7. **메시지 순서 보장 X** — 서버가 보장하지 않으면 client 측 sequence.

## 11. 다른 옵션 — Firebase Realtime Database / Firestore

```bash
yarn add @react-native-firebase/firestore
```

```ts
import firestore from '@react-native-firebase/firestore';

firestore()
  .collection('messages')
  .where('roomId', '==', 123)
  .orderBy('createdAt')
  .onSnapshot((snap) => {
    snap.docChanges().forEach((change) => {
      if (change.type === 'added') {
        // 새 메시지
      }
    });
  });
```

→ WebSocket / STOMP 없이 Firebase 가 알아서. 서버 구축 부담 없음. cost 발생.

## 12. 다음 단계

- [[../auth/auth]] — token + WebSocket
- [[../native-features/push-fcm]] — 백그라운드 메시지

## 13. 외부 자료

- [@stomp/stompjs](https://stomp-js.github.io/stomp-websocket/)
- [Socket.IO client](https://socket.io/docs/v4/client-api/)
- [react-native WebSocket](https://reactnative.dev/docs/network#websocket-support)

## 14. 관련

- [[networking]]
- [[axios-fetch]]
- [[../native-features/push-fcm]]
