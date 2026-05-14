---
title: "PG mock tests — WireMock + Sandbox"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T19:58:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - product
  - testing
  - pg
  - wiremock
---

# PG mock tests — WireMock + Sandbox

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v1.0.0 | 2026-05-14 | engineering-agent/tech-lead | 최초 |

**[[testing|↑ hub]]**

---

## 1. WireMock 셋업

```java
@AutoConfigureWireMock(port = 0)
class PaymentConfirmIT extends IntegrationTestBase {

    @Value("${wiremock.server.port}")
    int wireMockPort;

    @DynamicPropertySource
    static void props(DynamicPropertyRegistry r) {
        // PG base URL 을 WireMock 으로
        r.add("pg.toss.base-url", () -> "http://localhost:" + wireMockPort);
    }

    @Test void confirm_pg_returns_done() {
        stubFor(post(urlEqualTo("/v1/payments/confirm"))
            .willReturn(okJson("""
                {
                  "paymentKey": "pk_123",
                  "status": "DONE",
                  "approvedAt": "2026-05-14T18:30:00+09:00",
                  "card": { "company": "신한", "number": "************1234" }
                }
                """)));

        // when /confirm 호출
        // then payment DONE
    }
}
```

---

## 2. PG 실패 시나리오

```java
@Test void pg_timeout_results_in_fail() {
    stubFor(post(urlEqualTo("/v1/payments/confirm"))
        .willReturn(aResponse().withFixedDelay(10_000)));   // 10s 응답

    // server timeout 5s → PgException → payment FAILED
}

@Test void pg_returns_4xx_known_error() {
    stubFor(post(urlEqualTo("/v1/payments/confirm"))
        .willReturn(jsonResponse("""
            { "code": "ALREADY_PROCESSED_PAYMENT", "message": "..." }
            """, 400)));

    // payment FAILED with code ALREADY_PROCESSED
}
```

---

## 3. PG sandbox (실제 PG)

```yaml
# application-sandbox.yml
pg:
  toss:
    base-url: https://api.tosspayments.com
    secret-key: ${TOSS_SECRET_KEY}            # sandbox key
    client-key: ${TOSS_CLIENT_KEY}
```

→ `@Tag("sandbox")` test 만 실제 PG 호출. CI 는 WireMock.

---

## 4. 함정

### 함정 1 — 모든 테스트가 실 PG 호출
CI 느림 + sandbox 한도 소진.
→ WireMock 기본 / sandbox tagged only.

### 함정 2 — sandbox 와 production 동일 코드
sandbox 만의 quirk 가 production 에서 다름.
→ 출시 전 실 카드 (소액) 테스트.

---

## 5. 관련

- [[testing|↑ hub]]
- [[integration-tests]]
- [[../implementation/payment-confirm-impl]]
- [[../design-decisions/pg-selection]]
