---
title: "Adapter"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, structural, adapter, wrapper]
---

# Adapter

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Structural

## 1. 한 줄

**호환 안 되는 인터페이스를 클라이언트가 기대하는 인터페이스로 변환.** "변환기".

## 2. 언제 쓰는가

- **legacy / 3rd-party API 의 인터페이스가 우리 시스템과 안 맞음** — Stripe API 의 인터페이스를 사내 표준 PaymentGateway interface 에 맞춤.
- **여러 vendor 의 다른 API 를 같은 인터페이스로** — S3 / GCS / Azure Blob 의 storage abstraction.
- **legacy 코드 wrapping** — 옛 클래스를 새 인터페이스에.
- **테스트 시 mock 의 interface 맞춤**.

## 3. Adapter vs Facade vs Bridge

| | Adapter | Facade | Bridge |
| --- | --- | --- | --- |
| 목적 | **인터페이스 변환** | 단순 인터페이스 제공 | 추상/구현 분리 (설계 시) |
| 시점 | 사후 (incompatible 만남) | 사후 또는 설계 | 설계 |
| 대상 | 한 객체 | 여러 객체 / subsystem | 두 계층 |

## 4. 구조

```
Client → Target Interface
              ▲
              │ implements
              │
            Adapter ──── adaptee (legacy class) ──► uses
```

- **Object adapter** (composition): adaptee 를 멤버로.
- **Class adapter** (multiple inheritance): adaptee 를 상속.

## 5. Java / Spring

```java
// 우리 시스템의 인터페이스
public interface PaymentGateway {
    void pay(BigDecimal amount, String currency);
}

// 3rd-party (Stripe SDK)
public class StripeClient {
    public void chargeCustomer(int cents, String iso) { /* ... */ }
}

// Adapter
public class StripeAdapter implements PaymentGateway {
    private final StripeClient stripe;

    public StripeAdapter(StripeClient stripe) { this.stripe = stripe; }

    @Override
    public void pay(BigDecimal amount, String currency) {
        int cents = amount.multiply(BigDecimal.valueOf(100)).intValue();
        stripe.chargeCustomer(cents, currency.toUpperCase());
    }
}
```

**Spring**: 여러 PaymentGateway 구현체를 `@Qualifier` 로 골라 주입. Vendor 변경 = adapter 교체만.

**Spring 의 `HandlerAdapter`**: Servlet/Reactive/RestController 의 다양한 handler 를 같은 `DispatcherServlet` interface 로.

## 6. Python / Django / FastAPI

```python
class PaymentGateway(Protocol):
    def pay(self, amount: Decimal, currency: str) -> None: ...

# 3rd-party
class StripeClient:
    def charge_customer(self, cents: int, iso: str) -> None: ...

# Adapter
class StripeAdapter:
    def __init__(self, stripe: StripeClient):
        self.stripe = stripe

    def pay(self, amount: Decimal, currency: str) -> None:
        self.stripe.charge_customer(int(amount * 100), currency.upper())
```

**Django 의 backend abstraction**:
- `EMAIL_BACKEND = 'django.core.mail.backends.smtp.EmailBackend'` — SMTP, console, file 등 다른 backend = adapter.
- `DATABASES['ENGINE']` — postgresql / mysql / sqlite 의 adapter.

**FastAPI**: 외부 API client 를 `Depends` 통해 adapter 로 wrap.

## 7. TypeScript / NestJS / React

```ts
interface PaymentGateway {
  pay(amount: number, currency: string): Promise<void>;
}

// 3rd-party (Stripe SDK)
class StripeClient {
  async chargeCustomer(cents: number, iso: string): Promise<void> { /* ... */ }
}

class StripeAdapter implements PaymentGateway {
  constructor(private stripe: StripeClient) {}

  async pay(amount: number, currency: string) {
    await this.stripe.chargeCustomer(Math.round(amount * 100), currency.toUpperCase());
  }
}
```

**NestJS**: Provider 의 `useClass` 로 adapter 주입 — `{ provide: 'PAYMENT', useClass: StripeAdapter }`.

**React**: 3rd-party UI library 를 own component 로 wrap = adapter. `<MyButton/>` 이 내부에서 MUI Button + 자체 props.

## 8. Go

```go
type PaymentGateway interface {
    Pay(amount decimal.Decimal, currency string) error
}

// 3rd-party
type StripeClient struct{ /* ... */ }
func (s *StripeClient) ChargeCustomer(cents int, iso string) error { /* ... */ }

// Adapter
type StripeAdapter struct{ client *StripeClient }

func (a *StripeAdapter) Pay(amount decimal.Decimal, currency string) error {
    cents := amount.Mul(decimal.NewFromInt(100)).IntPart()
    return a.client.ChargeCustomer(int(cents), strings.ToUpper(currency))
}
```

**Go std**: `io.Reader` / `io.Writer` 가 만능 adapter. 어떤 source 든 `Read([]byte)` 만 구현하면 io 함수 다 통과.

## 9. 함정

1. **Adapter 가 거대 conversion 로직** — 진짜 mapping 이라기보다 비즈니스 로직이 섞임. service 로 분리.
2. **양방향 adapter** — 두 인터페이스 모두 만족해야 할 때. 추상화 깨짐 신호.
3. **2-Way wrapping (Adapter inside Adapter)** — 디버깅 지옥. 1 단계만.
4. **lossy conversion** — 한 인터페이스의 정보가 다른 쪽에 없음. 사일렌트 정보 손실 위험.

## 10. 관련

- [[../facade/facade]] — 단순 인터페이스.
- [[../bridge/bridge]] — 설계 시 분리.
- [[../decorator/decorator]] — 책임 추가 (인터페이스 변환 X).
- [[../proxy/proxy]] — 같은 인터페이스 대리.
