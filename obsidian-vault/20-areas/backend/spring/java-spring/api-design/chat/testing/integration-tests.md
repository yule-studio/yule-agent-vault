---
title: "Integration tests (WebSocketStompClient)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:16:00+09:00
tags: [backend, java-spring, api-design, chat, testing, integration]
---

# Integration tests (WebSocketStompClient)

**[[testing|↑ hub]]**

---

## 1. Base

```java
@SpringBootTest(webEnvironment = WebEnvironment.RANDOM_PORT)
@Testcontainers
abstract class ChatIntegrationTestBase {

    @LocalServerPort int port;

    @Container static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");
    @Container static GenericContainer<?> redis = new GenericContainer<>("redis:7")
        .withExposedPorts(6379);

    protected WebSocketStompClient stompClient() {
        var client = new StandardWebSocketClient();
        var stomp = new WebSocketStompClient(client);
        stomp.setMessageConverter(new MappingJackson2MessageConverter());
        return stomp;
    }

    protected StompSession connect(String jwt) throws Exception {
        var url = "ws://localhost:" + port + "/ws";
        var headers = new WebSocketHttpHeaders();
        var connectHeaders = new StompHeaders();
        connectHeaders.add("Authorization", "Bearer " + jwt);
        return stompClient().connectAsync(url, headers, connectHeaders,
            new StompSessionHandlerAdapter() {}).get(5, SECONDS);
    }
}
```

---

## 2. Send + Receive

```java
@Test void sendReceive() throws Exception {
    var aJwt = tokenFor(userA);
    var bJwt = tokenFor(userB);
    var roomId = roomService.createDirect(userA, userB).value();

    var sessionA = connect(aJwt);
    var sessionB = connect(bJwt);

    var received = new ArrayBlockingQueue<MessageDto>(10);
    sessionB.subscribe("/topic/room/" + roomId, new StompFrameHandler() {
        public Type getPayloadType(StompHeaders h) { return MessageDto.class; }
        public void handleFrame(StompHeaders h, Object p) { received.offer((MessageDto) p); }
    });

    Thread.sleep(200);   // subscribe ready

    sessionA.send("/app/room/" + roomId + "/send",
        Map.of("type", "TEXT", "content", "안녕"));

    var msg = received.poll(5, SECONDS);
    assertThat(msg).isNotNull();
    assertThat(msg.content()).isEqualTo("안녕");
}
```

---

## 3. Auth fail

```java
@Test void invalid_jwt_rejected() {
    var future = stompClient().connectAsync(...);
    assertThatThrownBy(future::get)
        .hasCauseInstanceOf(MessageDeliveryException.class);
}
```

---

## 4. 관련

- [[testing|↑ hub]]
- [[unit-tests]]
- [[load-tests]]
