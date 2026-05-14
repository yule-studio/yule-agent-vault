---
title: "E2E tests"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-15T02:20:00+09:00
tags: [backend, java-spring, api-design, chat, testing, e2e]
---

# E2E tests

**[[testing|↑ hub]]**

---

## Playwright (multi-user, multi-device)

```typescript
test('1:1 chat send/receive', async ({ browser }) => {
    // 2 사용자 (= 2 browser context)
    const ctxA = await browser.newContext();
    const ctxB = await browser.newContext();

    const pageA = await ctxA.newPage();
    const pageB = await ctxB.newPage();

    await loginAs(pageA, 'userA');
    await loginAs(pageB, 'userB');

    await pageA.goto('/chat/room-1');
    await pageB.goto('/chat/room-1');

    // A send
    await pageA.fill('[data-testid=message-input]', '안녕');
    await pageA.click('[data-testid=send]');

    // B receive
    await expect(pageB.locator('[data-testid=message-bubble]:last-child'))
        .toContainText('안녕');
});

test('multi-device sync', async ({ browser }) => {
    // 같은 user A 의 2 device (= 2 browser context, 같은 JWT)
    // pageA1 = phone, pageA2 = tablet
    // pageA1 에서 send → pageA2 에서도 보임
});

test('read receipt', async ({ browser }) => {
    // A send → "1" 표시
    // B 가 page 열음 → A 화면에서 "1" → "0" 변경
});
```

---

## 관련

- [[testing|↑ hub]]
- [[integration-tests]]
