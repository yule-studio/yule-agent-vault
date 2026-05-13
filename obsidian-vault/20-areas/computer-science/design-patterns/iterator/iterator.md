---
title: "Iterator"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, iterator, generator, lazy]
---

# Iterator

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**컬렉션 내부 구조 노출 없이 순회.** for-loop 의 기반.

## 2. 언제 쓰는가

- **컬렉션의 traversal API**.
- **lazy / 무한 sequence** — generator.
- **다양한 traversal 순서** (in-order / pre-order / BFS).
- **다른 collection type 의 통일 API**.

## 3. Java

```java
// 기본 — Iterable + Iterator
class Numbers implements Iterable<Integer> {
    private int[] data = {1, 2, 3, 4, 5};

    @Override
    public Iterator<Integer> iterator() {
        return new Iterator<>() {
            int idx = 0;
            public boolean hasNext() { return idx < data.length; }
            public Integer next() { return data[idx++]; }
        };
    }
}

// for-each = iterator 사용
Numbers n = new Numbers();
for (int x : n) System.out.println(x);
```

**Stream API** (Java 8+):
```java
List<Integer> nums = List.of(1, 2, 3, 4, 5);
nums.stream()
    .filter(x -> x % 2 == 0)
    .map(x -> x * 10)
    .forEach(System.out::println);
```

**Spring `JpaRepository`** 의 `findAll()` 이 Iterable.

## 4. Python — 가장 idiomatic

```python
class Numbers:
    def __init__(self):
        self.data = [1, 2, 3, 4, 5]
    def __iter__(self):
        return iter(self.data)

for x in Numbers():
    print(x)
```

**Generator function — 가장 pythonic**:
```python
def fib(n):
    a, b = 0, 1
    for _ in range(n):
        yield a
        a, b = b, a + b

for x in fib(10):
    print(x)
```

- `yield` 의 lazy evaluation.
- 메모리 효율 (전체 list 안 만듦).
- 무한 sequence 가능.

**Generator expression**:
```python
squares = (x*x for x in range(1_000_000))   # lazy
for s in squares: ...
```

**Django QuerySet** = lazy iterator:
```python
users = User.objects.filter(active=True)   # SQL 실행 안 됨
for u in users: print(u)                    # 여기서 chunked SQL
```

**FastAPI streaming response**:
```python
from fastapi.responses import StreamingResponse

def generate():
    for i in range(1000):
        yield f"data: {i}\n\n"

@app.get("/stream")
def stream(): return StreamingResponse(generate(), media_type="text/event-stream")
```

## 5. TypeScript

```ts
class Numbers implements Iterable<number> {
  private data = [1, 2, 3, 4, 5];

  [Symbol.iterator](): Iterator<number> {
    let i = 0;
    const data = this.data;
    return {
      next(): IteratorResult<number> {
        return i < data.length
          ? { value: data[i++], done: false }
          : { value: undefined as any, done: true };
      },
    };
  }
}

for (const x of new Numbers()) console.log(x);
```

**Generator**:
```ts
function* fib(n: number): Generator<number> {
  let a = 0, b = 1;
  for (let i = 0; i < n; i++) {
    yield a;
    [a, b] = [b, a + b];
  }
}

for (const x of fib(10)) console.log(x);
```

**Async iterator**:
```ts
async function* streamLines(url: string) {
  const res = await fetch(url);
  const reader = res.body!.getReader();
  // ...
  yield line;
}

for await (const line of streamLines('...')) console.log(line);
```

**RxJS Observable** = Iterator 의 push 버전.

## 6. Go

```go
// Go 1.22 까지: idiomatic 은 closure / channel
type Numbers struct{ data []int }

func (n *Numbers) Iter() func() (int, bool) {
    i := 0
    return func() (int, bool) {
        if i >= len(n.data) { return 0, false }
        v := n.data[i]; i++
        return v, true
    }
}

n := &Numbers{data: []int{1,2,3,4,5}}
next := n.Iter()
for {
    v, ok := next()
    if !ok { break }
    fmt.Println(v)
}
```

**Go 1.23+ — `range over function`**:
```go
func (n *Numbers) All() func(yield func(int) bool) {
    return func(yield func(int) bool) {
        for _, v := range n.data {
            if !yield(v) { return }
        }
    }
}

for v := range n.All() {
    fmt.Println(v)
}
```

- Go 1.23 의 `iter` package 와 `range` 가 함수 받기.
- standard iterator pattern.

**channel 기반 (옛 idiom)**:
```go
func generator(ctx context.Context) <-chan int {
    ch := make(chan int)
    go func() {
        defer close(ch)
        for i := 0; i < 100; i++ {
            select {
            case ch <- i:
            case <-ctx.Done(): return
            }
        }
    }()
    return ch
}

for v := range generator(ctx) {
    fmt.Println(v)
}
```

- goroutine leak 위험 — context 필수.

## 7. External vs Internal Iterator

- **External (pull)**: caller 가 `next()` 호출 — `for x in collection`.
- **Internal (push)**: collection 이 callback 호출 — `collection.each(callback)` (Ruby), `.forEach()`.

```js
// external
for (const x of arr) console.log(x);

// internal
arr.forEach(x => console.log(x));
```

- External 이 더 유연 (break / 부분 traversal).
- Internal 이 더 간결.

## 8. Tree / Graph Iterator

```python
def dfs(node):
    yield node
    for child in node.children:
        yield from dfs(child)

def bfs(root):
    queue = [root]
    while queue:
        node = queue.pop(0)
        yield node
        queue.extend(node.children)
```

## 9. 함정

1. **mutation during iteration** — `for x in list: list.append(...)` = undefined behavior. snapshot 또는 separate collection.
2. **iterator exhaustion** — Python generator 는 1 회 소비. 다시 iter 필요.
3. **lazy evaluation 의 side effect 시점** — Django QuerySet 의 SQL 이 언제 실행되는지 모호. `list(qs)` 로 강제 evaluate.
4. **무한 generator + `list()`** — 메모리 폭주.
5. **Go channel iterator 의 leak** — receiver 안 하면 sender goroutine 영원히 block. context cancellation.
6. **multi-threaded iteration** — Java `ConcurrentModificationException`. `Concurrent` collection 사용.

## 10. 관련

- [[../composite/composite]] — tree traversal.
- [[../visitor/visitor]] — collection 에 연산 적용.
- [[../observer/observer]] — push 기반 (iterator 는 pull).
