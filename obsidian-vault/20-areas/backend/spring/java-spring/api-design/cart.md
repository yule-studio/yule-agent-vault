---
title: "장바구니 (Cart) — Java Spring Boot 레시피 (쿠팡 / 무신사 스타일)"
kind: knowledge
project: backend
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-14T14:00:00+09:00
tags:
  - backend
  - java-spring
  - api-design
  - recipe
  - commerce
  - cart
---

# 장바구니 (Cart) — Java Spring Boot 레시피 (쿠팡 / 무신사 스타일)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-14 | engineering-agent/tech-lead | guest Redis + member RDB merge / 가격 스냅샷 / 재고 검증 |

**[[api-design|↑ api-design hub]]**

> 전제: [[signup]] · [[login-jwt]] · [[product-crud]]. `users`, `products`, `product_options` 존재.
> **참고**: 쿠팡 (회원·비회원 통합 카트), 무신사 (옵션별 카트), 당근 (단일 상품 대화).

---

## 1. 무엇을 만드는가

```
# 회원
GET    /api/v1/cart                          # 내 카트 조회
POST   /api/v1/cart/items                    # 항목 추가
PATCH  /api/v1/cart/items/{itemId}           # 수량 변경
DELETE /api/v1/cart/items/{itemId}           # 항목 삭제
POST   /api/v1/cart/clear                    # 전체 비우기
POST   /api/v1/cart/merge                    # 게스트 카트 흡수 (로그인 직후)

# 비회원 (게스트)
GET    /api/v1/cart?guestId={uuid}           # 또는 cookie 기반
POST   /api/v1/cart/items?guestId={uuid}
...
```

### 1.1 요청 / 응답 — 카트 조회

```http
GET /api/v1/cart
Authorization: Bearer ...

200 OK
{
  "data": {
    "items": [
      {
        "itemId": "01HZCART...",
        "productId": "01HZPRD...",
        "productName": "오버사이즈 후드 — 차콜",
        "optionId": "01HZOPT...",
        "optionLabel": "Size: M",
        "unitPriceKrw": 49000,
        "quantity": 2,
        "lineTotalKrw": 98000,
        "mainImageUrl": "https://cdn.../1.jpg",
        "available": true,
        "issues": []
      }
    ],
    "subtotalKrw": 98000,
    "issueCount": 0,
    "updatedAt": "2026-05-14T14:05:00Z"
  }
}
```

`issues` 예: `OUT_OF_STOCK`, `PRICE_CHANGED`, `PRODUCT_DISCONTINUED`. 결제 직전에 다시 검증.

### 1.2 비기능

- **회원**: 영속 (RDB). 디바이스 간 동기화.
- **비회원**: Redis TTL (예: 30일). guest UUID 발급.
- **로그인 시 merge** — guest 카트 → member 카트로 흡수.
- **가격 스냅샷 X** (담을 때 가격 저장 X) — 결제 직전에 현재가로 재계산. UX 로 "가격이 바뀌었습니다" 안내.
- **재고는 hold X** — 카트는 의도 표현일 뿐. 진짜 hold 는 [[order-stock]] 의 주문 생성 시점.

---

## 2. 도메인 모델

### 2.1 Cart Aggregate

```java
// src/main/java/com/example/shop/domain/cart/Cart.java
public final class Cart {

    private final CartId id;
    private final CartOwner owner;                  // member 또는 guest
    private final List<CartItem> items = new ArrayList<>();
    private final Instant createdAt;
    private Instant updatedAt;
    private final List<DomainEvent> events = new ArrayList<>();

    private static final int MAX_ITEMS = 100;
    private static final int MAX_QTY_PER_ITEM = 99;

    private Cart(CartId id, CartOwner owner, Instant createdAt, Instant updatedAt) {
        this.id = id; this.owner = owner;
        this.createdAt = createdAt; this.updatedAt = updatedAt;
    }

    public static Cart open(CartId id, CartOwner owner, Instant now) {
        return new Cart(id, owner, now, now);
    }

    public CartItem addItem(CartItemId itemId, ProductId productId, ProductOptionId optionId,
                            int quantity, Instant now) {
        if (quantity <= 0 || quantity > MAX_QTY_PER_ITEM)
            throw new IllegalArgumentException("invalid qty: " + quantity);

        // 같은 product+option 이 이미 있으면 수량만 증가
        var existing = items.stream()
            .filter(i -> i.productId().equals(productId) && i.optionId().equals(optionId))
            .findFirst();

        if (existing.isPresent()) {
            existing.get().addQuantity(quantity, MAX_QTY_PER_ITEM);
            updatedAt = now;
            return existing.get();
        }

        if (items.size() >= MAX_ITEMS) throw new IllegalStateException("cart full");

        var item = CartItem.create(itemId, productId, optionId, quantity, now);
        items.add(item);
        updatedAt = now;
        events.add(new CartItemAdded(id, item.id(), productId, optionId, quantity, now));
        return item;
    }

    public void updateQuantity(CartItemId itemId, int newQty, Instant now) {
        if (newQty <= 0 || newQty > MAX_QTY_PER_ITEM)
            throw new IllegalArgumentException("invalid qty");
        var item = findItem(itemId);
        item.setQuantity(newQty);
        updatedAt = now;
        events.add(new CartItemUpdated(id, itemId, newQty, now));
    }

    public void removeItem(CartItemId itemId, Instant now) {
        var removed = items.removeIf(i -> i.id().equals(itemId));
        if (!removed) throw new NotFoundException();
        updatedAt = now;
        events.add(new CartItemRemoved(id, itemId, now));
    }

    public void clear(Instant now) {
        items.clear();
        updatedAt = now;
        events.add(new CartCleared(id, now));
    }

    /** guest cart 의 item 들을 흡수. 같은 product+option 은 수량 합산. */
    public void absorb(Cart guestCart, Instant now) {
        if (!owner.isMember()) throw new IllegalStateException("only member cart can absorb");
        for (var gi : guestCart.items()) {
            addItem(new CartItemId(ULID.random()),
                    gi.productId(), gi.optionId(), gi.quantity(), now);
        }
    }

    private CartItem findItem(CartItemId id) {
        return items.stream().filter(i -> i.id().equals(id)).findFirst()
            .orElseThrow(NotFoundException::new);
    }

    public CartId id() { return id; }
    public CartOwner owner() { return owner; }
    public List<CartItem> items() { return List.copyOf(items); }
    public Instant createdAt() { return createdAt; }
    public Instant updatedAt() { return updatedAt; }

    public List<DomainEvent> pullDomainEvents() {
        var copy = List.copyOf(events); events.clear(); return copy;
    }
}
```

### 2.2 CartItem

```java
// src/main/java/com/example/shop/domain/cart/CartItem.java
public final class CartItem {
    private final CartItemId id;
    private final ProductId productId;
    private final ProductOptionId optionId;
    private int quantity;
    private final Instant addedAt;

    private CartItem(CartItemId id, ProductId pid, ProductOptionId oid, int qty, Instant addedAt) {
        this.id = id; this.productId = pid; this.optionId = oid;
        this.quantity = qty; this.addedAt = addedAt;
    }

    public static CartItem create(CartItemId id, ProductId pid, ProductOptionId oid,
                                  int qty, Instant addedAt) {
        return new CartItem(id, pid, oid, qty, addedAt);
    }

    void setQuantity(int newQty) { this.quantity = newQty; }
    void addQuantity(int delta, int max) {
        var sum = quantity + delta;
        if (sum > max) throw new IllegalArgumentException("qty exceeds " + max);
        this.quantity = sum;
    }

    public CartItemId id() { return id; }
    public ProductId productId() { return productId; }
    public ProductOptionId optionId() { return optionId; }
    public int quantity() { return quantity; }
    public Instant addedAt() { return addedAt; }
}
```

### 2.3 CartOwner — sealed type

```java
// src/main/java/com/example/shop/domain/cart/CartOwner.java
public sealed interface CartOwner permits MemberOwner, GuestOwner {
    boolean isMember();
    String key();           // Redis / DB lookup key
}

public record MemberOwner(UserId userId) implements CartOwner {
    public boolean isMember() { return true; }
    public String key() { return "u:" + userId.value(); }
}

public record GuestOwner(String guestId) implements CartOwner {
    public GuestOwner {
        if (guestId == null || guestId.length() < 16)
            throw new IllegalArgumentException("invalid guestId");
    }
    public boolean isMember() { return false; }
    public String key() { return "g:" + guestId; }
}
```

### 2.4 Repository — 두 구현 (RDB / Redis)

```java
// domain/cart/CartRepository.java
public interface CartRepository {
    Optional<Cart> findByOwner(CartOwner owner);
    Cart save(Cart cart);
    void delete(CartOwner owner);
}
```

---

## 3. 저장 전략 — RDB (회원) vs Redis (게스트)

### 3.1 비교

| | 회원 (RDB) | 게스트 (Redis) |
| --- | --- | --- |
| 영속 | ✅ | ⚠️ TTL 30일 |
| 디바이스 동기화 | ✅ (계정 기준) | ❌ (브라우저별 guestId) |
| 검색 / 분석 | ✅ (DW 적재) | ⚠️ |
| 쓰기 빈도 | 보통 | 매우 잦음 |
| latency | ms | μs |
| 비용 | DB 영속 | RAM |

→ **회원 = RDB / 게스트 = Redis** 가 표준. 사용자 ID 결정에 따라 repository 분기.

### 3.2 Adapter 패턴

```java
// src/main/java/com/example/shop/infrastructure/persistence/cart/CompositeCartRepository.java
@Repository
@Primary
public class CompositeCartRepository implements CartRepository {

    private final JpaCartRepositoryAdapter memberRepo;
    private final RedisCartRepository guestRepo;

    public CompositeCartRepository(JpaCartRepositoryAdapter memberRepo,
                                   RedisCartRepository guestRepo) {
        this.memberRepo = memberRepo; this.guestRepo = guestRepo;
    }

    @Override
    public Optional<Cart> findByOwner(CartOwner owner) {
        return switch (owner) {
            case MemberOwner m -> memberRepo.findByOwner(m);
            case GuestOwner g  -> guestRepo.findByOwner(g);
        };
    }

    @Override
    public Cart save(Cart cart) {
        return switch (cart.owner()) {
            case MemberOwner m -> memberRepo.save(cart);
            case GuestOwner g  -> guestRepo.save(cart);
        };
    }

    @Override
    public void delete(CartOwner owner) {
        switch (owner) {
            case MemberOwner m -> memberRepo.delete(owner);
            case GuestOwner g  -> guestRepo.delete(owner);
        }
    }
}
```

> Java 21 의 sealed + switch pattern matching 으로 깔끔.

---

## 4. DB / Redis 스키마

### 4.1 RDB — 회원

```sql
-- V30__create_carts.sql
CREATE TABLE carts (
  id          CHAR(26) PRIMARY KEY,
  user_id     CHAR(26) NOT NULL REFERENCES users(id),
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ux_carts_user ON carts (user_id);

-- V31__create_cart_items.sql
CREATE TABLE cart_items (
  id          CHAR(26) PRIMARY KEY,
  cart_id     CHAR(26) NOT NULL REFERENCES carts(id) ON DELETE CASCADE,
  product_id  CHAR(26) NOT NULL REFERENCES products(id),
  option_id   CHAR(26) NOT NULL REFERENCES product_options(id),
  quantity    INTEGER  NOT NULL CHECK (quantity > 0),
  added_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ix_cart_items_cart ON cart_items (cart_id);
CREATE UNIQUE INDEX ux_cart_items_unique ON cart_items (cart_id, product_id, option_id);
```

> **함정**: `cart_items` 의 `(product_id, option_id)` unique 가 없으면 같은 옵션이 두 줄로 들어감. 도메인 로직 외에 DB 가 진실의 원천.

### 4.2 Redis — 게스트

key 형식: `cart:g:{guestId}` — Hash 또는 JSON 직렬화.

```
cart:g:GUEST-01HZ-XXXX        (Hash)
  {item-01HZ-A}  → {"productId":"...","optionId":"...","quantity":2,"addedAt":"..."}
  {item-01HZ-B}  → {"productId":"...","optionId":"...","quantity":1,"addedAt":"..."}
TTL: 30d
```

JSON 한 덩어리로 저장하는 변형도 가능 (간단 / 갱신 시 전체 쓰기). 항목 100개 이내면 한 덩어리도 OK.

### 4.3 Redis 구현

```java
// src/main/java/com/example/shop/infrastructure/persistence/cart/RedisCartRepository.java
@Component
public class RedisCartRepository implements CartRepository {

    private final StringRedisTemplate redis;
    private final ObjectMapper json;
    private final Clock clock;
    private static final Duration TTL = Duration.ofDays(30);

    public RedisCartRepository(StringRedisTemplate redis, ObjectMapper json, Clock clock) {
        this.redis = redis; this.json = json; this.clock = clock;
    }

    @Override
    public Optional<Cart> findByOwner(CartOwner owner) {
        if (!(owner instanceof GuestOwner g)) throw new IllegalArgumentException();
        var key = key(g);
        var raw = redis.opsForValue().get(key);
        if (raw == null) return Optional.empty();
        return Optional.of(deserialize(raw, g));
    }

    @Override
    public Cart save(Cart cart) {
        if (!(cart.owner() instanceof GuestOwner g)) throw new IllegalArgumentException();
        var key = key(g);
        var raw = serialize(cart);
        redis.opsForValue().set(key, raw, TTL);
        return cart;
    }

    @Override
    public void delete(CartOwner owner) {
        if (!(owner instanceof GuestOwner g)) throw new IllegalArgumentException();
        redis.delete(key(g));
    }

    private String key(GuestOwner g) { return "cart:" + g.key(); }

    private String serialize(Cart c) { /* JSON 직렬화 — items, timestamps */ throw new UnsupportedOperationException(); }
    private Cart deserialize(String s, GuestOwner g) { /* 역직렬화 + Cart.reconstitute */ throw new UnsupportedOperationException(); }
}
```

---

## 5. UseCase

### 5.1 AddCartItemUseCase

```java
// src/main/java/com/example/shop/application/cart/AddCartItemUseCase.java
@Service
public class AddCartItemUseCase {

    private final CartRepository carts;
    private final ProductRepository products;
    private final IdGenerator ids;
    private final Clock clock;
    private final ApplicationEventPublisher events;

    public AddCartItemUseCase(CartRepository carts, ProductRepository products,
                              IdGenerator ids, Clock clock, ApplicationEventPublisher events) {
        this.carts = carts; this.products = products;
        this.ids = ids; this.clock = clock; this.events = events;
    }

    @Transactional
    public CartItemId handle(CartOwner owner, ProductId productId,
                             ProductOptionId optionId, int quantity) {
        // 상품 / 옵션 검증
        var product = products.findById(productId)
            .orElseThrow(() -> new NotFoundException("product"));
        if (product.status() != ProductStatus.ACTIVE)
            throw new ProductNotAvailableException(product.status());
        var option = product.options().stream()
            .filter(o -> o.id().equals(optionId))
            .findFirst()
            .orElseThrow(() -> new NotFoundException("option"));
        // 재고 hold X — 안내만
        if (option.stock() < quantity)
            throw new InsufficientStockException(optionId, quantity, option.stock());

        var cart = carts.findByOwner(owner)
            .orElseGet(() -> Cart.open(new CartId(ids.next()), owner, Instant.now(clock)));

        var item = cart.addItem(
            new CartItemId(ids.next()),
            productId, optionId, quantity, Instant.now(clock)
        );

        var saved = carts.save(cart);
        saved.pullDomainEvents().forEach(events::publishEvent);
        return item.id();
    }
}
```

### 5.2 MergeGuestCartUseCase

```java
// src/main/java/com/example/shop/application/cart/MergeGuestCartUseCase.java
@Service
public class MergeGuestCartUseCase {

    private final CartRepository carts;
    private final IdGenerator ids;
    private final Clock clock;

    public MergeGuestCartUseCase(CartRepository carts, IdGenerator ids, Clock clock) {
        this.carts = carts; this.ids = ids; this.clock = clock;
    }

    @Transactional
    public void handle(UserId userId, String guestId) {
        var guestOwner = new GuestOwner(guestId);
        var guest = carts.findByOwner(guestOwner);
        if (guest.isEmpty()) return;                    // 합칠 것 없음

        var memberOwner = new MemberOwner(userId);
        var member = carts.findByOwner(memberOwner)
            .orElseGet(() -> Cart.open(new CartId(ids.next()), memberOwner, Instant.now(clock)));

        try {
            member.absorb(guest.get(), Instant.now(clock));
        } catch (IllegalStateException e) {
            // MAX_ITEMS 초과 등 — 부분 흡수 정책 또는 사용자 안내
            throw new CartMergeOverflowException(e.getMessage());
        }
        carts.save(member);

        // 게스트 카트 삭제 — 같은 사람이 다시 비회원으로 돌아가도 동기화 X
        carts.delete(guestOwner);
    }
}
```

### 5.3 ViewCartUseCase (가격 / 재고 / 상태 검증)

```java
@Service
public class ViewCartUseCase {

    private final CartRepository carts;
    private final ProductRepository products;

    public ViewCartUseCase(CartRepository carts, ProductRepository products) {
        this.carts = carts; this.products = products;
    }

    @Transactional(readOnly = true)
    public CartView view(CartOwner owner) {
        var cart = carts.findByOwner(owner).orElse(emptyFor(owner));
        var lines = new ArrayList<CartView.Line>();
        long subtotal = 0;
        int issueCount = 0;

        for (var it : cart.items()) {
            var maybeProduct = products.findById(it.productId());
            if (maybeProduct.isEmpty()) {
                lines.add(unavailable(it, "PRODUCT_NOT_FOUND"));
                issueCount++;
                continue;
            }
            var product = maybeProduct.get();
            if (product.status() == ProductStatus.DISCONTINUED
                || product.status() == ProductStatus.DELETED) {
                lines.add(unavailable(it, "PRODUCT_DISCONTINUED"));
                issueCount++;
                continue;
            }
            var maybeOption = product.options().stream()
                .filter(o -> o.id().equals(it.optionId())).findFirst();
            if (maybeOption.isEmpty()) {
                lines.add(unavailable(it, "OPTION_NOT_FOUND"));
                issueCount++;
                continue;
            }
            var option = maybeOption.get();
            var unitPrice = product.basePrice().add(option.extraPrice());
            var issues = new ArrayList<String>();
            if (option.stock() < it.quantity()) issues.add("OUT_OF_STOCK");
            if (issues.isEmpty()) subtotal += unitPrice.amountMinorUnit() * it.quantity();
            else issueCount++;

            lines.add(new CartView.Line(
                it.id().value(), it.productId().value(), product.name(),
                it.optionId().value(), option.name() + ": " + option.value(),
                unitPrice.amountMinorUnit(), it.quantity(),
                unitPrice.amountMinorUnit() * it.quantity(),
                "https://cdn/x.jpg",       // mainImageUrl - product 에서
                issues.isEmpty(), issues
            ));
        }

        return new CartView(lines, subtotal, issueCount, cart.updatedAt());
    }
}
```

> **핵심**: 가격은 항상 **현재 상품 가격** 으로 재계산. 카트에 가격 저장 X.

---

## 6. Controller

```java
// src/main/java/com/example/shop/presentation/api/v1/cart/CartController.java
@RestController
@RequestMapping("/api/v1/cart")
public class CartController {

    private final ViewCartUseCase view;
    private final AddCartItemUseCase add;
    private final UpdateCartItemUseCase updateQty;
    private final RemoveCartItemUseCase remove;
    private final MergeGuestCartUseCase merge;

    public CartController(ViewCartUseCase view, AddCartItemUseCase add,
                          UpdateCartItemUseCase updateQty, RemoveCartItemUseCase remove,
                          MergeGuestCartUseCase merge) {
        this.view = view; this.add = add; this.updateQty = updateQty;
        this.remove = remove; this.merge = merge;
    }

    @GetMapping
    public ApiResponse<CartView> get(
        @AuthenticationPrincipal(expression = "this") Object authPrincipal,
        @RequestParam(required = false) String guestId
    ) {
        return ApiResponse.ok(view.view(resolveOwner(authPrincipal, guestId)));
    }

    @PostMapping("/items")
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<Map<String, String>> add(
        @Valid @RequestBody AddItemRequest req,
        @RequestParam(required = false) String guestId,
        Authentication auth
    ) {
        var owner = resolveOwner(auth, guestId);
        var itemId = add.handle(owner, new ProductId(req.productId()),
                                new ProductOptionId(req.optionId()), req.quantity());
        return ApiResponse.ok(Map.of("itemId", itemId.value()));
    }

    @PatchMapping("/items/{itemId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void update(@PathVariable String itemId,
                       @Valid @RequestBody UpdateItemRequest req,
                       @RequestParam(required = false) String guestId,
                       Authentication auth) {
        updateQty.handle(resolveOwner(auth, guestId), new CartItemId(itemId), req.quantity());
    }

    @DeleteMapping("/items/{itemId}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable String itemId,
                       @RequestParam(required = false) String guestId,
                       Authentication auth) {
        remove.handle(resolveOwner(auth, guestId), new CartItemId(itemId));
    }

    @PostMapping("/merge")
    @PreAuthorize("isAuthenticated()")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void merge(@Valid @RequestBody MergeRequest req, Authentication auth) {
        merge.handle(new UserId(auth.getName()), req.guestId());
    }

    private static CartOwner resolveOwner(Object auth, String guestId) {
        if (auth instanceof Authentication a && a.isAuthenticated()
            && !"anonymousUser".equals(a.getPrincipal())) {
            return new MemberOwner(new UserId(a.getName()));
        }
        if (guestId == null) throw new BadRequestException("guestId required");
        return new GuestOwner(guestId);
    }
}

public record AddItemRequest(
    @NotBlank String productId,
    @NotBlank String optionId,
    @Min(1) @Max(99) int quantity
) {}
public record UpdateItemRequest(@Min(1) @Max(99) int quantity) {}
public record MergeRequest(@NotBlank String guestId) {}
```

### 6.1 게스트 ID 발급

```java
// 첫 접속 시 cookie 로 발급
@Component
public class GuestIdResolver implements HandlerInterceptor {
    @Override
    public boolean preHandle(HttpServletRequest req, HttpServletResponse res, Object handler) {
        var existing = Arrays.stream(req.getCookies() == null ? new Cookie[0] : req.getCookies())
            .filter(c -> "GID".equals(c.getName()))
            .findFirst();
        if (existing.isEmpty()) {
            var newId = "g_" + ULID.random();
            var c = new Cookie("GID", newId);
            c.setHttpOnly(true);
            c.setSecure(true);
            c.setMaxAge(60 * 60 * 24 * 30);   // 30d
            c.setPath("/");
            c.setAttribute("SameSite", "Lax");
            res.addCookie(c);
            req.setAttribute("guestId", newId);
        } else {
            req.setAttribute("guestId", existing.get().getValue());
        }
        return true;
    }
}
```

> 쿠팡 / 무신사 식 — 비회원도 카트 가능, cookie 로 추적, 로그인 시 merge.

---

## 7. 로그인 흐름과 통합

로그인 시점에 merge 호출 — Controller 가 아니라 LoginUseCase 가 도메인 이벤트로 트리거하거나, 클라이언트가 명시적으로 `/merge` 호출.

```
1. 사용자가 비회원으로 카트 추가 (cookie GID=g_xxx)
2. 로그인 → access/refresh token 발급
3. 클라이언트가 POST /api/v1/cart/merge { guestId: g_xxx }
4. 서버: guest 카트 → member 카트 흡수 + guest 카트 삭제
5. 클라이언트: GID cookie 만료
```

→ **명시적 merge endpoint** 권장. 자동 merge 는 사용자 의도와 다를 수 있음 (다른 사람의 PC 등).

---

## 8. 테스트

### 8.1 도메인

```java
@Test
void adding_same_product_option_increments_quantity_instead_of_new_item() {
    var cart = Cart.open(new CartId(ulid()), new MemberOwner(new UserId(ulid())), Instant.now());
    cart.addItem(new CartItemId(ulid()), pid, oid, 2, Instant.now());
    cart.addItem(new CartItemId(ulid()), pid, oid, 3, Instant.now());
    assertThat(cart.items()).hasSize(1);
    assertThat(cart.items().get(0).quantity()).isEqualTo(5);
}

@Test
void member_cart_absorbs_guest_cart_items() {
    var member = Cart.open(new CartId(ulid()), new MemberOwner(uid1), Instant.now());
    member.addItem(new CartItemId(ulid()), pid, oid1, 1, Instant.now());
    var guest = Cart.open(new CartId(ulid()), new GuestOwner("g_"+ulid()), Instant.now());
    guest.addItem(new CartItemId(ulid()), pid, oid1, 2, Instant.now());
    guest.addItem(new CartItemId(ulid()), pid, oid2, 1, Instant.now());

    member.absorb(guest, Instant.now());

    assertThat(member.items()).hasSize(2);
    assertThat(member.items().stream().filter(i -> i.optionId().equals(oid1)).findFirst()
        .orElseThrow().quantity()).isEqualTo(3);   // 1 + 2
}
```

### 8.2 통합 — guest → 로그인 → merge

```java
@Test
void guest_cart_is_merged_into_member_cart_on_login() {
    // guest 추가
    rest.postForEntity("/api/v1/cart/items?guestId=g_test",
        Map.of("productId", pid, "optionId", oid, "quantity", 2), Map.class);

    // 회원 가입 + 로그인
    var user = userFixture.createActive("u@x.com", "p@ss12word");
    var access = loginAndGetAccess("u@x.com", "p@ss12word");

    // merge
    var h = new HttpHeaders(); h.setBearerAuth(access);
    rest.exchange("/api/v1/cart/merge", HttpMethod.POST,
        new HttpEntity<>(Map.of("guestId", "g_test"), h), Void.class);

    // 회원 카트 조회
    var res = rest.exchange("/api/v1/cart", HttpMethod.GET,
        new HttpEntity<>(h), Map.class);
    var items = (List<?>) ((Map<?, ?>) res.getBody().get("data")).get("items");
    assertThat(items).hasSize(1);
}
```

---

## 9. 운영 체크리스트

- [ ] 게스트 cookie `HttpOnly + Secure + SameSite=Lax`
- [ ] Redis 키 TTL 30일 (게스트만)
- [ ] 카트 최대 항목 / 최대 수량 제한 (DoS 방어)
- [ ] 가격 / 재고 / 상태 검증을 카트 조회 시점에 매번
- [ ] 가격 변경 시 사용자 안내 (`issues: ["PRICE_CHANGED"]`)
- [ ] DISCONTINUED / DELETED 상품은 자동 제거 X — 사용자가 의식적으로 삭제
- [ ] merge 시 항목 수 한계 초과 처리 정책 (앞부분만 / 거절)
- [ ] 게스트 카트의 데이터셋 별도 (Redis instance / 격리)
- [ ] 카트 조회 endpoint p95 < 100ms 모니터
- [ ] 보안: guestId 는 cookie 만 — query param 직접 받지 말 것 (URL 공유 시 카트 공유됨)

---

## 10. 함정 모음

### 함정 1 — 카트에 가격 / 상품명 저장
스냅샷 모델 = 가격 변동 시 카트와 실제 가격 불일치. 결제 시점이 진실. 단, 분쟁 위해 **주문 (Order) 에는 스냅샷 필수** ([[order-stock]]).

### 함정 2 — 재고 hold
카트는 의도 표현. 진짜 hold 는 주문 생성 / 결제 진입 시점. 카트에서 hold = 재고 부족 + 카트 방치 = 다른 사용자 구매 못함.

### 함정 3 — `(cart_id, product_id, option_id)` unique 누락
같은 옵션이 두 줄. 표시 / 합계 깨짐. **DB unique 필수**.

### 함정 4 — `guestId` 를 query parameter
로그 / URL 공유로 카트 탈취. **HttpOnly cookie 만**.

### 함정 5 — 로그인 시 자동 merge
다른 사람 PC 에서 로그인하면 그 사람의 guest 카트가 내 카트에 섞임. **명시적 merge endpoint + 사용자 확인 UX**.

### 함정 6 — Redis 영속 오해
RDB AOF 만 있어도 fail 시 데이터 손실 가능. **게스트 카트는 잃어도 OK** 정책으로 운영.

### 함정 7 — DISCONTINUED 상품 카트에서 자동 제거
사용자가 "왜 사라졌지?" — UX 혼란. **표시는 하되 구매 불가 표시** + 사용자 결정.

### 함정 8 — 카트 endpoint 가 `count(*)` 매번
카트 항목 100개 이내라면 무관. 그래도 매번 Repository 호출 N+1 회피.

### 함정 9 — Redis serialization 의 timestamp 형식 변경
스키마 진화 시 옛 데이터 역직렬화 실패. 버전 필드 + 마이그레이션 함수.

### 함정 10 — 사용자가 다른 디바이스에서 동시 수정
A 디바이스 +1, B 디바이스 -1 동시 — last-write-wins 으로 한쪽 사라짐. 보통은 허용 (UX 영향 적음). 엄격 필요 시 `@Version` + 재시도.

---

## 11. 관련

- [[product-crud]] · [[product-search]]
- [[order-stock]] (다음) — 카트 → 주문 변환 + 재고 hold
- [[payment-pg]] (다음) — 주문 → 결제
- [[../pitfalls/n-plus-one]] — Cart view 의 N+1 회피
- [[api-design|↑ api-design hub]]
