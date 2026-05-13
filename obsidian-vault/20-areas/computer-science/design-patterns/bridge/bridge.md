---
title: "Bridge"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, structural, bridge]
---

# Bridge

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Structural

## 1. 한 줄

**추상화 (interface) 와 구현 (implementation) 을 두 차원으로 분리.** 둘이 독립적으로 확장.

## 2. 언제 쓰는가

- **속성 × 행동의 클래스 폭발** — Shape × Color = N×M 개 class. Bridge 로 N+M 으로.
- **런타임에 구현 swap** — 같은 추상화의 backend 를 환경 별로.
- **API 와 platform 분리** — UI framework 가 OS 위 abstraction layer.

## 3. Bridge vs Adapter

- **Bridge**: **설계 시** 두 차원을 분리. 같은 author 가 둘 다 디자인.
- **Adapter**: **사후** incompatible interface 변환. 다른 author 의 라이브러리.

## 4. 구조

```
Abstraction        ──has─►    Implementor (interface)
  + op()                          + opImpl()
  ▲                                ▲
  │                                │
RefinedAbstraction      ConcreteImpA  ConcreteImpB
```

- Abstraction 변경 / Implementor 변경 독립.

## 5. 예시 시나리오

```
Shape (Abstraction)          Renderer (Implementor)
  Circle, Square                SvgRenderer, CanvasRenderer

조합:
  Circle + SvgRenderer
  Square + CanvasRenderer
  ...

Shape × Renderer = N + M classes (NOT N×M).
```

## 6. Java

```java
interface Renderer {
    void renderCircle(double r);
    void renderSquare(double s);
}

class SvgRenderer implements Renderer {
    public void renderCircle(double r) { /* svg */ }
    public void renderSquare(double s) { /* svg */ }
}

class CanvasRenderer implements Renderer {
    public void renderCircle(double r) { /* canvas */ }
    public void renderSquare(double s) { /* canvas */ }
}

abstract class Shape {
    protected Renderer renderer;        // bridge
    Shape(Renderer r) { this.renderer = r; }
    abstract void draw();
}

class Circle extends Shape {
    double radius;
    Circle(Renderer r, double rad) { super(r); this.radius = rad; }
    void draw() { renderer.renderCircle(radius); }
}

class Square extends Shape {
    double side;
    Square(Renderer r, double s) { super(r); this.side = s; }
    void draw() { renderer.renderSquare(side); }
}
```

**Spring 예**: `LoggingProvider` (실제 logging API — Logback / Log4j / java.util.logging) 와 `Logger` (호출 면) 의 분리. SLF4J 가 정확히 Bridge 패턴.

## 7. Python

```python
from abc import ABC, abstractmethod

class Renderer(ABC):
    @abstractmethod
    def render_circle(self, r): ...
    @abstractmethod
    def render_square(self, s): ...

class SvgRenderer:
    def render_circle(self, r): print(f"<circle r='{r}'/>")
    def render_square(self, s): print(f"<rect w='{s}'/>")

class CanvasRenderer:
    def render_circle(self, r): pass  # canvas API
    def render_square(self, s): pass

class Shape:
    def __init__(self, renderer): self.renderer = renderer
    def draw(self): raise NotImplementedError

class Circle(Shape):
    def __init__(self, renderer, r): super().__init__(renderer); self.r = r
    def draw(self): self.renderer.render_circle(self.r)
```

**Django**: ORM 의 backend abstraction — `DATABASES['ENGINE']` 가 PostgreSQL / MySQL / SQLite 의 구체 구현 (Bridge), `Model` API 가 추상화 면.

**Python `logging`** = Bridge — `Logger` (abstraction) + `Handler` (implementation, FileHandler / StreamHandler / SysLogHandler).

## 8. TypeScript / React

```ts
interface Renderer {
  renderCircle(r: number): void;
  renderSquare(s: number): void;
}

class SvgRenderer implements Renderer {
  renderCircle(r: number) { /* svg */ }
  renderSquare(s: number) { /* svg */ }
}

class CanvasRenderer implements Renderer {
  renderCircle(r: number) { /* canvas */ }
  renderSquare(s: number) { /* canvas */ }
}

abstract class Shape {
  constructor(protected renderer: Renderer) {}
  abstract draw(): void;
}

class Circle extends Shape {
  constructor(renderer: Renderer, public radius: number) { super(renderer); }
  draw() { this.renderer.renderCircle(this.radius); }
}
```

**React**: theme system — `ThemeProvider` (implementation) + 컴포넌트 (abstraction). 같은 `<Button/>` 이 dark / light 다르게.

**React Native vs React Web**: 같은 component API + 다른 renderer = Bridge.

## 9. Go

```go
type Renderer interface {
    RenderCircle(r float64)
    RenderSquare(s float64)
}

type svgRenderer struct{}
func (svgRenderer) RenderCircle(r float64) { /* ... */ }
func (svgRenderer) RenderSquare(s float64) { /* ... */ }

type Shape interface {
    Draw()
}

type Circle struct {
    r        float64
    renderer Renderer
}
func (c Circle) Draw() { c.renderer.RenderCircle(c.r) }
```

**Go std**: `io.Writer` interface = 모든 output 의 abstraction. `os.File`, `bytes.Buffer`, `net.Conn` 모두 같은 Writer 인터페이스로 추상화 — Bridge 의 광범위 사용.

## 10. 함정

1. **Bridge 가 필요 없는 단순 case 에 적용** — 한 implementation 만이면 그냥 직접 사용. YAGNI.
2. **N + M 가 N×M 이 되어버림** — Bridge 의 추상화 면이 implementation 면에 의존하면 본래 의도 실패.
3. **circular dependency** — Abstraction ↔ Implementor.
4. **너무 일찍 분리** — 첫 implementation 나오기 전 추상화하면 wrong abstraction.

## 11. 관련

- [[../adapter/adapter]] — 사후 변환.
- [[../abstract-factory/abstract-factory]] — Bridge 의 Implementor family 생성.
- [[../strategy/strategy]] — 비슷한 구조, 다른 의도 (Bridge = 설계, Strategy = 알고리즘 교체).
