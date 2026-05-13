---
title: "Facade"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, structural, facade]
---

# Facade

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Structural

## 1. 한 줄

**복잡한 subsystem 위에 단순한 interface 한 개를 얹는다.**

## 2. 언제 쓰는가

- **legacy / 복잡 subsystem 의 사용자 친화 API**.
- **monolithic 모듈을 한 진입점으로**.
- **library / SDK 의 단순 API** — 내부는 복잡한 여러 client.
- **마이크로서비스 BFF (Backend for Frontend)** — 여러 backend service 를 frontend 가 쓰기 좋게 한 진입점.

## 3. Facade vs Adapter

- **Facade**: 단순 인터페이스 제공 (1 → N). 새 API 제공.
- **Adapter**: 한 인터페이스를 다른 인터페이스로 변환 (1 → 1).

## 4. 구조

```
Client → Facade ──► subsystem class A
                 ├► subsystem class B
                 ├► subsystem class C
                 └► subsystem class D
```

## 5. Java / Spring

```java
// 복잡 subsystem
class StockService { Stock get(String id) { ... } }
class PricingService { Price calc(Stock s) { ... } }
class OrderRepository { void save(Order o) { ... } }
class NotificationService { void notify(User u, Order o) { ... } }

// Facade
@Service
public class OrderFacade {
    @Autowired private StockService stock;
    @Autowired private PricingService pricing;
    @Autowired private OrderRepository repo;
    @Autowired private NotificationService notify;

    public Order placeOrder(User u, List<String> productIds) {
        List<Stock> items = productIds.stream().map(stock::get).toList();
        Price total = pricing.calc(items);
        Order o = new Order(u, items, total);
        repo.save(o);
        notify.notify(u, o);
        return o;
    }
}

// Client
@RestController
public class OrderController {
    @Autowired private OrderFacade orders;

    @PostMapping("/order")
    public Order order(@RequestBody OrderReq req) {
        return orders.placeOrder(req.user, req.productIds);
    }
}
```

**Spring** 의 모든 `@Service` 가 사실상 Facade — 여러 repository / external service 위 단순 method.

## 6. Python / Django / FastAPI

```python
class OrderFacade:
    def __init__(self, stock, pricing, repo, notify):
        self.stock = stock
        self.pricing = pricing
        self.repo = repo
        self.notify = notify

    def place_order(self, user, product_ids):
        items = [self.stock.get(pid) for pid in product_ids]
        total = self.pricing.calc(items)
        order = Order(user=user, items=items, total=total)
        self.repo.save(order)
        self.notify.notify(user, order)
        return order

# Django view
def place_order_view(request):
    facade = OrderFacade(...)
    order = facade.place_order(request.user, request.POST.getlist('product_id'))
    return JsonResponse(model_to_dict(order))
```

**Django ORM `Manager`**: `User.objects` 가 Facade — 내부 QuerySet / SQL compiler / database adapter / connection pool 모두 hide.

**Django `django.shortcuts.render`**: HttpResponse + template engine + context processor + middleware chain 의 facade.

**FastAPI `Depends`** + service class 가 보통 Facade 역할.

## 7. TypeScript / NestJS / React

```ts
// NestJS
@Injectable()
export class OrderFacade {
  constructor(
    private stock: StockService,
    private pricing: PricingService,
    private repo: OrderRepository,
    private notify: NotificationService,
  ) {}

  async placeOrder(user: User, productIds: string[]): Promise<Order> {
    const items = await Promise.all(productIds.map(id => this.stock.get(id)));
    const total = this.pricing.calc(items);
    const order = new Order(user, items, total);
    await this.repo.save(order);
    await this.notify.notify(user, order);
    return order;
  }
}

@Controller('orders')
export class OrderController {
  constructor(private orders: OrderFacade) {}

  @Post()
  place(@Body() dto: OrderDto, @CurrentUser() user: User) {
    return this.orders.placeOrder(user, dto.productIds);
  }
}
```

**React custom hook 이 Facade**:
```tsx
function useUser() {
  const auth = useAuth();
  const profile = useProfile(auth.userId);
  const permissions = usePermissions(auth.userId);
  return { auth, profile, permissions };
}

// Component 가 단순화
function Header() {
  const { profile } = useUser();
  return <span>{profile.name}</span>;
}
```

**Frontend BFF** — Node/NestJS 가 여러 microservice 호출을 frontend 한 API 로 facade.

## 8. Go

```go
type OrderFacade struct {
    Stock    StockService
    Pricing  PricingService
    Repo     OrderRepository
    Notify   NotificationService
}

func (f *OrderFacade) PlaceOrder(user User, productIDs []string) (*Order, error) {
    items := make([]Stock, 0, len(productIDs))
    for _, id := range productIDs {
        s, err := f.Stock.Get(id)
        if err != nil { return nil, err }
        items = append(items, s)
    }
    total := f.Pricing.Calc(items)
    order := &Order{User: user, Items: items, Total: total}
    if err := f.Repo.Save(order); err != nil { return nil, err }
    f.Notify.Notify(user, order)
    return order, nil
}
```

**gRPC / HTTP server handler** 가 보통 Facade 역할 — request 받아 여러 service 조립.

## 9. 함정

1. **Facade 가 거대 god object** — 한 service 안 모든 비즈니스 로직 — Single Responsibility 위반.
2. **Facade 가 단순한 wrapper** — 가치 없는 1:1 위임. 그냥 subsystem 직접.
3. **모든 subsystem call 을 facade 강제** — 일부 client 는 subsystem 직접 필요. Facade 가 hard rule 이면 깨짐.
4. **Facade vs Service vs Controller 혼동** — 책임 명확:
   - Controller: HTTP / 외부 입력 처리.
   - Facade / Service: 비즈니스 로직 조립.
   - Repository: 데이터 access.
5. **트랜잭션 경계의 모호함** — `@Transactional` 을 facade method 에 두면 subsystem 사이 트랜잭션 일관성 보장. 명시적 설계.

## 10. 관련

- [[../adapter/adapter]] — 인터페이스 변환.
- [[../mediator/mediator]] — 다대다 중개.
- [[../proxy/proxy]] — 같은 인터페이스 대리.
