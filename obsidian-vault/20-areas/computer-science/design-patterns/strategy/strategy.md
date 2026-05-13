---
title: "Strategy"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, strategy, algorithm, polymorphism]
---

# Strategy

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**알고리즘 family 를 인터페이스로 캡슐화. 런타임에 교체.**

## 2. 언제 쓰는가

- **같은 일의 여러 방법** — 결제 (Stripe/PayPal/Toss), 인증 (Local/OAuth/SSO), 정렬 (quick/merge).
- **if-else 폭주** — 5+ 개 분기.
- **테스트 친화** — 실제 vs fake / mock.
- **plug-in** — 외부에서 알고리즘 등록.

## 3. Strategy vs State vs Template Method

| | Strategy | State | Template Method |
| --- | --- | --- | --- |
| 교체 | 외부 명시적 | 객체 내부 자동 | 상속 |
| 시점 | 런타임 | 런타임 (전이 시) | 컴파일 |
| 알고리즘 골격 | 다양 | 다양 | 고정 + 일부 step 자식 |

## 4. 구조

```
Context                     Strategy (interface)
  - strategy: Strategy        + execute(data)
  + setStrategy(s)            ▲
  + doWork() {                │
      strategy.execute()      ConcreteStrategyA  B  C
    }
```

## 5. Java / Spring

```java
interface PaymentStrategy {
    void pay(BigDecimal amount, User user);
}

class StripePayment implements PaymentStrategy {
    public void pay(BigDecimal a, User u) { /* Stripe API */ }
}

class PaypalPayment implements PaymentStrategy {
    public void pay(BigDecimal a, User u) { /* PayPal API */ }
}

class CheckoutService {
    private PaymentStrategy strategy;
    public void setStrategy(PaymentStrategy s) { this.strategy = s; }
    public void checkout(BigDecimal amount, User u) {
        strategy.pay(amount, u);
    }
}
```

**Spring 의 자동 DI + Strategy**:
```java
@Service
class CheckoutService {
    private final Map<String, PaymentStrategy> strategies;

    public CheckoutService(List<PaymentStrategy> all) {
        this.strategies = all.stream()
            .collect(toMap(PaymentStrategy::getName, identity()));
    }

    public void checkout(String type, BigDecimal amount, User u) {
        strategies.get(type).pay(amount, u);
    }
}

@Component class StripePayment implements PaymentStrategy { ... }
@Component class PaypalPayment implements PaymentStrategy { ... }
// Spring 이 모든 PaymentStrategy 구현체 자동 주입
```

**Spring Security `AuthenticationProvider`** — Strategy chain.

## 6. Python / Django

```python
from typing import Protocol
from decimal import Decimal

class PaymentStrategy(Protocol):
    def pay(self, amount: Decimal, user: User) -> None: ...

class StripePayment:
    def pay(self, amount, user): pass  # Stripe API

class PaypalPayment:
    def pay(self, amount, user): pass  # PayPal API

class CheckoutService:
    def __init__(self, strategy: PaymentStrategy):
        self.strategy = strategy
    def checkout(self, amount, user):
        self.strategy.pay(amount, user)

# Functional — 함수 일급 객체
def stripe_pay(amount, user): pass
def paypal_pay(amount, user): pass

class CheckoutFunc:
    def __init__(self, pay_fn): self.pay = pay_fn
    def checkout(self, amount, user):
        self.pay(amount, user)
```

→ Python 의 함수 일급 객체 → strategy = 함수 자체.

**Django authentication backends** = Strategy:
```python
# settings.py
AUTHENTICATION_BACKENDS = [
    'django.contrib.auth.backends.ModelBackend',
    'allauth.account.auth_backends.AuthenticationBackend',
    'myapp.auth.LDAPBackend',
]
```

**Django Storage backend** / **Email backend** / **Cache backend** 모두 strategy.

**Django REST Framework `permission_classes`** = strategy list.

## 7. TypeScript / NestJS / React

```ts
interface PaymentStrategy {
  pay(amount: number, user: User): Promise<void>;
}

class StripePayment implements PaymentStrategy {
  async pay(amount: number, user: User) { /* ... */ }
}

class PaypalPayment implements PaymentStrategy {
  async pay(amount: number, user: User) { /* ... */ }
}

class CheckoutService {
  constructor(private strategy: PaymentStrategy) {}
  async checkout(amount: number, user: User) {
    await this.strategy.pay(amount, user);
  }
}
```

**NestJS Provider + Strategy registry**:
```ts
@Module({
  providers: [
    StripePayment,
    PaypalPayment,
    {
      provide: 'PAYMENT_STRATEGIES',
      useFactory: (s: StripePayment, p: PaypalPayment) => ({
        stripe: s,
        paypal: p,
      }),
      inject: [StripePayment, PaypalPayment],
    },
  ],
})
export class PaymentModule {}
```

**React — strategy = component / prop**:
```tsx
function Sort<T>({ data, strategy }: { data: T[]; strategy: (a: T, b: T) => number }) {
  const sorted = [...data].sort(strategy);
  return <ul>{sorted.map(...)}</ul>;
}

<Sort data={users} strategy={(a, b) => a.name.localeCompare(b.name)} />
```

**Functional strategy** = 함수 자체:
```ts
type Validator = (value: string) => boolean;

const isEmail: Validator = v => /@/.test(v);
const isPhone: Validator = v => /^\d{3}-\d{4}-\d{4}$/.test(v);

function validate(value: string, strategy: Validator) {
  return strategy(value);
}
```

## 8. Go

```go
type PaymentStrategy interface {
    Pay(amount decimal.Decimal, user User) error
}

type StripePayment struct{}
func (s *StripePayment) Pay(amount decimal.Decimal, user User) error { /* ... */ return nil }

type PaypalPayment struct{}
func (p *PaypalPayment) Pay(amount decimal.Decimal, user User) error { /* ... */ return nil }

type CheckoutService struct{ Strategy PaymentStrategy }

func (c *CheckoutService) Checkout(amount decimal.Decimal, user User) error {
    return c.Strategy.Pay(amount, user)
}

// 사용
svc := &CheckoutService{Strategy: &StripePayment{}}
svc.Checkout(decimal.NewFromInt(100), user)
```

**Function strategy** — Go 의 first-class function:
```go
type PaymentFunc func(amount decimal.Decimal, user User) error

func stripePay(amount decimal.Decimal, user User) error { /* ... */ return nil }
func paypalPay(amount decimal.Decimal, user User) error { /* ... */ return nil }

type Service struct{ Pay PaymentFunc }

svc := &Service{Pay: stripePay}
svc.Pay(decimal.NewFromInt(100), user)
```

**Go `sort.Slice`** = strategy:
```go
sort.Slice(users, func(i, j int) bool { return users[i].Age < users[j].Age })
sort.Slice(users, func(i, j int) bool { return users[i].Name < users[j].Name })
```

## 9. Strategy + Factory

여러 strategy 중 선택:

```python
class StrategyRegistry:
    _strategies = {}

    @classmethod
    def register(cls, name): 
        def decorator(strategy_cls):
            cls._strategies[name] = strategy_cls
            return strategy_cls
        return decorator

    @classmethod
    def get(cls, name) -> PaymentStrategy:
        return cls._strategies[name]()

@StrategyRegistry.register("stripe")
class StripePayment: ...

@StrategyRegistry.register("paypal")
class PaypalPayment: ...

strategy = StrategyRegistry.get("stripe")
```

## 10. 함정

1. **Strategy class 폭발** — 5+ strategy 정상, 100+ 면 다시 검토.
2. **Strategy 사이 shared state** — 어떤 strategy 가 다른 strategy 의 state 의존 시 race.
3. **Strategy 와 Context 의 결합** — strategy 가 Context 의 private 에 의존하면 강결합.
4. **Strategy interface 의 bloat** — strategy 마다 다른 추가 메서드 강요. Interface Segregation.
5. **테스트 시 wrong strategy 주입** — DI container 의 default 가 production strategy. test 에서 별도 inject 필수.
6. **functional strategy 의 디버깅** — closure 내부 추적 어려움. named function 권장.

## 11. 관련

- [[../state/state]] — 비슷한 구조, 다른 의도 (자동 전환).
- [[../template-method/template-method]] — 상속 기반 알고리즘 골격.
- [[../command/command]] — 알고리즘 + 결과 객체화.
- [[../factory-method/factory-method]] — strategy 의 생성.
