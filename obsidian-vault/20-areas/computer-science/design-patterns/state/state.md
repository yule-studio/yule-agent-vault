---
title: "State"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, state, fsm, statemachine]
---

# State

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**상태 별 행동을 별도 객체로.** if-elif 폭주 회피 + 상태 전이가 명시적.

## 2. State vs Strategy

- **Strategy**: 외부가 알고리즘 명시적 교체.
- **State**: 객체가 내부에서 자동 전환.
- 코드 구조는 매우 비슷.

## 3. 언제 쓰는가

- **상태 전이가 명확한 객체** — 주문 (CREATED → PAID → SHIPPED → DELIVERED).
- **State 별 가능 동작이 다름** — 출입문 (LOCKED 면 open() 거부).
- **state 가 늘어남** — if/elif chain → 분리.
- **state machine / workflow** — Camunda / AWS Step Functions / Temporal.

## 4. 구조

```
Context                    State (interface)
  - state: State            + handle(context)
  + setState(s)
  + request() {              ▲
      state.handle(this)     │
    }                        │
                    ConcreteStateA  ConcreteStateB
                    (각자 다른 행동 + 다음 state 결정)
```

## 5. Java

```java
interface OrderState {
    void pay(Order o);
    void ship(Order o);
    void deliver(Order o);
}

class CreatedState implements OrderState {
    public void pay(Order o) {
        System.out.println("Paying...");
        o.setState(new PaidState());
    }
    public void ship(Order o) { throw new IllegalStateException("not paid"); }
    public void deliver(Order o) { throw new IllegalStateException("not paid"); }
}

class PaidState implements OrderState {
    public void pay(Order o) { throw new IllegalStateException("already paid"); }
    public void ship(Order o) {
        System.out.println("Shipping...");
        o.setState(new ShippedState());
    }
    public void deliver(Order o) { throw new IllegalStateException("not shipped"); }
}

class ShippedState implements OrderState {
    public void pay(Order o) { throw new IllegalStateException("already paid"); }
    public void ship(Order o) { throw new IllegalStateException("already shipped"); }
    public void deliver(Order o) {
        System.out.println("Delivered");
        o.setState(new DeliveredState());
    }
}

class DeliveredState implements OrderState {
    public void pay(Order o) { throw new IllegalStateException("done"); }
    public void ship(Order o) { throw new IllegalStateException("done"); }
    public void deliver(Order o) { throw new IllegalStateException("done"); }
}

class Order {
    private OrderState state = new CreatedState();
    public void setState(OrderState s) { this.state = s; }
    public void pay() { state.pay(this); }
    public void ship() { state.ship(this); }
    public void deliver() { state.deliver(this); }
}
```

**Spring Statemachine** library = production-grade state machine:
```java
@Configuration
@EnableStateMachine
public class OrderStateMachineConfig extends StateMachineConfigurerAdapter<OrderStates, OrderEvents> {
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderStates, OrderEvents> tr) throws Exception {
        tr.withExternal().source(CREATED).target(PAID).event(PAY)
          .and().withExternal().source(PAID).target(SHIPPED).event(SHIP);
    }
}
```

## 6. Python / Django

```python
from abc import ABC, abstractmethod

class OrderState(ABC):
    @abstractmethod
    def pay(self, order): ...

class CreatedState(OrderState):
    def pay(self, order):
        print("paying")
        order.state = PaidState()

class PaidState(OrderState):
    def pay(self, order):
        raise ValueError("already paid")

class Order:
    def __init__(self):
        self.state = CreatedState()
    def pay(self):
        self.state.pay(self)

# 사용
o = Order()
o.pay()    # CreatedState 의 pay → PaidState 로 전이
o.pay()    # ValueError
```

**Django state machine library**: `django-fsm`:
```python
from django_fsm import FSMField, transition

class Order(models.Model):
    state = FSMField(default='created')

    @transition(field=state, source='created', target='paid')
    def pay(self):
        # transition logic
        pass

    @transition(field=state, source='paid', target='shipped')
    def ship(self):
        pass

# 사용
order = Order.objects.get(pk=1)
order.pay()      # transition
order.save()
```

## 7. TypeScript / NestJS / React

```ts
interface OrderState {
  pay(order: Order): void;
  ship(order: Order): void;
}

class CreatedState implements OrderState {
  pay(o: Order) {
    console.log('paying');
    o.setState(new PaidState());
  }
  ship() { throw new Error('not paid'); }
}

class PaidState implements OrderState {
  pay() { throw new Error('already paid'); }
  ship(o: Order) {
    console.log('shipping');
    o.setState(new ShippedState());
  }
}

class ShippedState implements OrderState {
  pay() { throw new Error('done'); }
  ship() { throw new Error('done'); }
}

class Order {
  private state: OrderState = new CreatedState();
  setState(s: OrderState) { this.state = s; }
  pay() { this.state.pay(this); }
  ship() { this.state.ship(this); }
}
```

**XState** — JS state machine library:
```ts
import { createMachine } from 'xstate';

const orderMachine = createMachine({
  id: 'order',
  initial: 'created',
  states: {
    created: { on: { PAY: 'paid' } },
    paid: { on: { SHIP: 'shipped' } },
    shipped: { on: { DELIVER: 'delivered' } },
    delivered: { type: 'final' },
  },
});
```

**React 의 useReducer** = state machine:
```tsx
const [state, dispatch] = useReducer(orderReducer, { status: 'created' });

function orderReducer(state, action) {
  switch (state.status) {
    case 'created':
      if (action.type === 'PAY') return { status: 'paid' };
      throw new Error();
    case 'paid':
      if (action.type === 'SHIP') return { status: 'shipped' };
      throw new Error();
    // ...
  }
}
```

**NestJS Workflow** / **Temporal Workflow Engine** = production state machine.

## 8. Go

```go
type OrderState interface {
    Pay(o *Order) error
    Ship(o *Order) error
    Deliver(o *Order) error
}

type createdState struct{}
func (createdState) Pay(o *Order) error {
    fmt.Println("paying")
    o.state = paidState{}
    return nil
}
func (createdState) Ship(o *Order) error    { return errors.New("not paid") }
func (createdState) Deliver(o *Order) error { return errors.New("not paid") }

type paidState struct{}
func (paidState) Pay(o *Order) error     { return errors.New("paid") }
func (paidState) Ship(o *Order) error {
    o.state = shippedState{}
    return nil
}
func (paidState) Deliver(o *Order) error { return errors.New("not shipped") }

type Order struct{ state OrderState }

func NewOrder() *Order { return &Order{state: createdState{}} }
func (o *Order) Pay() error     { return o.state.Pay(o) }
func (o *Order) Ship() error    { return o.state.Ship(o) }
func (o *Order) Deliver() error { return o.state.Deliver(o) }
```

**Kubernetes controllers** = state machine reconciler.

**looplab/fsm Go library**:
```go
fsm := fsm.NewFSM(
    "created",
    fsm.Events{
        {Name: "pay", Src: []string{"created"}, Dst: "paid"},
        {Name: "ship", Src: []string{"paid"}, Dst: "shipped"},
    },
    fsm.Callbacks{...},
)
fsm.Event(ctx, "pay")
```

## 9. State + Event Sourcing

```
event log:                state replay:
- OrderCreated            created state
- OrderPaid       ─►      paid state
- OrderShipped            shipped state
```

- Event 들이 state 의 history.
- 현 state = event 들의 fold.
- Memento + State 의 결합.

## 10. 함정

1. **state 간 cyclic transition** — 무한 loop / 모호한 다음 state.
2. **state 가 외부 의존성 보유** — state object 가 큰 service 보유 → testability ↓.
3. **state 클래스 폭주** — 상태 5+ 면 OK, 50+ 면 다시 if-elif 가 나을 수도.
4. **DB persist 시 state 클래스 직렬화** — state object 자체를 DB 에 못 저장. enum + lookup table 권장.
5. **transition logic 의 분산** — 어떤 transition 이 가능한지 한 곳에서 보기 어려움. 표 / dot diagram 권장.
6. **distributed state machine** — 여러 service 사이 state 동기화 = Saga / Workflow Engine 영역.

## 11. 관련

- [[../strategy/strategy]] — 비슷한 구조, 외부 교체.
- [[../command/command]] — state 전이를 command 로 표현.
- [[../memento/memento]] — state snapshot.
