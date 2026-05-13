---
title: "Flyweight"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, structural, flyweight, sharing, memory]
---

# Flyweight

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Structural

## 1. 한 줄

**공유로 메모리 절약.** 같은 데이터를 가진 인스턴스를 1 개만 만들고 재사용.

## 2. 언제 쓰는가

- **수많은 작은 객체 (수십만 +)** — 게임의 입자, 텍스트의 글자, 트리 잎.
- **객체의 일부 상태가 동일** — 같은 텍스처 / 같은 모델.
- **메모리 limit ↓** — embedded / 모바일 / 큰 graph.

## 3. Intrinsic vs Extrinsic 상태

- **Intrinsic** (불변): 공유 가능. 객체에 박힘 (예: 글자의 폰트, 색).
- **Extrinsic** (variable): 호출 시 외부에서 전달 (예: 글자의 위치 x,y).

```
   Flyweight (intrinsic 만 보유)
   + operation(extrinsic)
```

## 4. 구조

```
FlyweightFactory
  - pool: Map<key, Flyweight>
  + get(key) {
      if (not in pool) pool[key] = new Flyweight(key);
      return pool[key];
    }
```

## 5. Java

```java
import java.util.*;

// 글자 (Flyweight)
class CharacterStyle {
    final String font;        // intrinsic
    final int size;
    final String color;

    private CharacterStyle(String f, int s, String c) {
        font = f; size = s; color = c;
    }

    static Map<String, CharacterStyle> pool = new HashMap<>();

    static CharacterStyle of(String font, int size, String color) {
        String key = font + size + color;
        return pool.computeIfAbsent(key, k -> new CharacterStyle(font, size, color));
    }
}

// Document 의 각 char 는 style + 위치 만 보유
class CharInstance {
    char c;
    int x, y;                     // extrinsic
    CharacterStyle style;         // 공유

    CharInstance(char c, int x, int y, CharacterStyle s) {
        this.c = c; this.x = x; this.y = y; this.style = s;
    }
}

// 100만 chars 가 100 개 unique style 공유
```

**Java `Integer.valueOf(int)`** 의 cache (-128 ~ 127) = Flyweight. 같은 값의 Integer 객체 재사용.

**Java `String` literal pool** = Flyweight. 같은 literal 의 String 객체 1 개.

## 6. Python

```python
class CharacterStyle:
    _pool = {}

    def __new__(cls, font, size, color):
        key = (font, size, color)
        if key not in cls._pool:
            instance = super().__new__(cls)
            instance.font = font
            instance.size = size
            instance.color = color
            cls._pool[key] = instance
        return cls._pool[key]

# 사용
s1 = CharacterStyle("Arial", 12, "black")
s2 = CharacterStyle("Arial", 12, "black")
assert s1 is s2     # 같은 인스턴스
```

**Python `sys.intern(str)`** — string interning. 같은 string 1 개만 메모리.

**Python `int` cache** — small int (-5 ~ 256) 가 항상 같은 인스턴스. `a is b` 가 작은 정수에 True.

**Django ORM `_meta`**: Model class 마다 하나의 `_meta` 가 공유 — 모든 instance 가 같은 metadata 참조.

## 7. TypeScript / React

```ts
class StyleFactory {
  private static pool = new Map<string, Style>();

  static get(font: string, size: number, color: string): Style {
    const key = `${font}|${size}|${color}`;
    let s = this.pool.get(key);
    if (!s) {
      s = new Style(font, size, color);
      this.pool.set(key, s);
    }
    return s;
  }
}
```

**React 의 reconciliation**:
- 같은 type + 같은 key 의 element 는 재사용 (memoization).
- `React.memo` 가 props 같으면 component instance 재사용.
- virtual DOM 의 element 도 Flyweight 비슷.

**JSON.stringify cache** / **Lodash `_.memoize`** 도 Flyweight 사상.

## 8. Go

```go
package style

import "sync"

type Style struct {
    Font  string
    Size  int
    Color string
}

var (
    pool   = make(map[string]*Style)
    poolMu sync.RWMutex
)

func Get(font string, size int, color string) *Style {
    key := fmt.Sprintf("%s|%d|%s", font, size, color)

    poolMu.RLock()
    if s, ok := pool[key]; ok {
        poolMu.RUnlock()
        return s
    }
    poolMu.RUnlock()

    poolMu.Lock()
    defer poolMu.Unlock()
    if s, ok := pool[key]; ok { return s }  // double-check

    s := &Style{Font: font, Size: size, Color: color}
    pool[key] = s
    return s
}
```

**Go `sync.Pool`**: 다른 패턴이지만 비슷한 사상. 임시 객체 재사용.

```go
var bufPool = sync.Pool{
    New: func() interface{} { return new(bytes.Buffer) },
}

func handler(w http.ResponseWriter, r *http.Request) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer bufPool.Put(buf)
    buf.Reset()
    // ...
}
```

## 9. 게임 / 그래픽 예시

- **Minecraft 의 블록**: 수십억 블록이지만 같은 type 의 블록은 같은 model/texture 공유.
- **GPU instancing**: 같은 mesh 를 다른 transform 으로 수만 번 그림 (Flyweight + extrinsic transform).
- **글자 렌더링**: 같은 glyph + 폰트 caching.

## 10. 함정

1. **share 가 부적합한 mutable state** — flyweight 가 변하면 모든 사용처 영향. **intrinsic 은 반드시 immutable**.
2. **pool 메모리 누수** — 한 번 만든 flyweight 가 영원히 살아남음. weak reference (`WeakHashMap` / `WeakValueDictionary`) 권장.
3. **key hashing 비용** — 적은 unique 값에 비싼 hash. 단순 key 권장.
4. **너무 작은 메모리 절약** — flyweight overhead (lookup, pool 자체 메모리) 가 절약보다 크면 net loss.
5. **multi-threaded pool 의 race** — RWMutex / double-check.

## 11. 관련

- [[../singleton/singleton]] — 1 인스턴스 vs flyweight 의 N 공유.
- [[../prototype/prototype]] — 복제 vs 공유.
- [[../factory-method/factory-method]] — flyweight 의 factory.
