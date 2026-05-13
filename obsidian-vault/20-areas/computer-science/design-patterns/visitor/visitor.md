---
title: "Visitor"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, visitor, double-dispatch, ast]
---

# Visitor

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**객체 구조와 그 위 연산을 분리.** 새 연산 추가가 쉬움 (새 Visitor). 구조 변경은 모든 Visitor 영향 (대가).

## 2. Double Dispatch

OOP single dispatch (객체 타입에 따라 메서드 선택) 한계 극복:
- `node.accept(visitor)` → `visitor.visit_xxx(node)` 두 번 dispatch.
- 자식 type + visitor type 의 조합으로 행동 결정.

## 3. 언제 쓰는가

- **AST traversal** — 컴파일러 / 인터프리터 / linter / formatter.
- **다양한 연산 + 안정된 구조** — 새 분석 (lint / format / type check) 추가가 잦음.
- **graph / tree 의 여러 알고리즘** — DFS / BFS / 통계.

## 4. 언제 안 쓰는가

- **구조가 자주 변경됨** — 새 ConcreteElement = 모든 Visitor 수정.
- **함수형 / pattern matching 있는 언어** — `match` 표현이 더 자연스러움.
- **단일 연산** — visitor 가 overkill.

## 5. Visitor의 Expression Problem

| | OOP (subclass + method) | Visitor |
| --- | --- | --- |
| 새 element type 추가 | ✓ 쉬움 | ✗ 모든 visitor 수정 |
| 새 연산 추가 | ✗ 모든 element 수정 | ✓ 새 visitor class |

→ Element 안정 / 연산 자주 추가 → Visitor. 반대면 OOP 직접.

## 6. Java

```java
interface Node {
    <R> R accept(Visitor<R> v);
}

class NumberNode implements Node {
    final double value;
    public NumberNode(double v) { this.value = v; }
    public <R> R accept(Visitor<R> v) { return v.visitNumber(this); }
}

class AddNode implements Node {
    final Node left, right;
    public AddNode(Node l, Node r) { this.left = l; this.right = r; }
    public <R> R accept(Visitor<R> v) { return v.visitAdd(this); }
}

interface Visitor<R> {
    R visitNumber(NumberNode n);
    R visitAdd(AddNode n);
}

class Evaluator implements Visitor<Double> {
    public Double visitNumber(NumberNode n) { return n.value; }
    public Double visitAdd(AddNode n) {
        return n.left.accept(this) + n.right.accept(this);
    }
}
```

**실 사례**:
- **Java Compiler API** (`javax.tools.TreeVisitor`) — javac AST traversal.
- **ASM bytecode library** (`ClassVisitor`, `MethodVisitor`) — bytecode 분석.
- **JaCoCo / SpotBugs / Checkstyle** — visitor 기반.

## 7. Python — `singledispatchmethod` 또는 `match`

```python
from functools import singledispatchmethod

class Evaluator:
    @singledispatchmethod
    def visit(self, node): raise NotImplementedError

    @visit.register
    def _(self, n: NumberNode): return n.v

    @visit.register
    def _(self, n: AddNode): return self.visit(n.left) + self.visit(n.right)
```

또는 **`match` (Python 3.10+)** — 함수형 대안:
```python
def evaluate(node):
    match node:
        case NumberNode(v=v): return v
        case AddNode(left=l, right=r): return evaluate(l) + evaluate(r)
```

**Python `ast` 모듈** — Python AST 의 표준 visitor:
```python
import ast

class FunctionCounter(ast.NodeVisitor):
    def __init__(self): self.count = 0
    def visit_FunctionDef(self, node):
        self.count += 1
        self.generic_visit(node)

tree = ast.parse(open('file.py').read())
v = FunctionCounter(); v.visit(tree)
```

**Django ORM** 의 SQL compiler 가 query AST 위 visitor.

## 8. TypeScript

```ts
interface Visitor<R> {
  visitNumber(n: NumberNode): R;
  visitAdd(n: AddNode): R;
}

interface Node {
  accept<R>(v: Visitor<R>): R;
}

class NumberNode implements Node {
  constructor(public value: number) {}
  accept<R>(v: Visitor<R>): R { return v.visitNumber(this); }
}

class AddNode implements Node {
  constructor(public left: Node, public right: Node) {}
  accept<R>(v: Visitor<R>): R { return v.visitAdd(this); }
}

class Evaluator implements Visitor<number> {
  visitNumber(n: NumberNode) { return n.value; }
  visitAdd(n: AddNode) { return n.left.accept(this) + n.right.accept(this); }
}
```

**Discriminated union + switch** = TypeScript 의 더 idiomatic 대안:
```ts
type Node =
  | { kind: 'number'; value: number }
  | { kind: 'add'; left: Node; right: Node };

function evaluate(n: Node): number {
  switch (n.kind) {
    case 'number': return n.value;
    case 'add':    return evaluate(n.left) + evaluate(n.right);
  }
}
```

**TypeScript Compiler API**:
```ts
import * as ts from 'typescript';

function visit(node: ts.Node) {
  if (ts.isFunctionDeclaration(node)) console.log(node.name?.text);
  ts.forEachChild(node, visit);
}
```

**Babel AST traversal**, **ESLint rule** 모두 visitor.

## 9. Go — type switch (visitor 대신)

```go
type Node interface { node() }

type NumberNode struct{ Value float64 }
func (NumberNode) node() {}

type AddNode struct{ Left, Right Node }
func (AddNode) node() {}

func Evaluate(n Node) float64 {
    switch v := n.(type) {
    case NumberNode: return v.Value
    case AddNode:    return Evaluate(v.Left) + Evaluate(v.Right)
    default:         panic("unknown")
    }
}
```

**Go `go/ast` library** — 표준 visitor:
```go
import "go/ast"

type funcCounter struct{ count int }

func (f *funcCounter) Visit(n ast.Node) ast.Visitor {
    if _, ok := n.(*ast.FuncDecl); ok { f.count++ }
    return f
}

ast.Walk(&funcCounter{}, file)
```

**golangci-lint / staticcheck / gopls** 모두 ast visitor.

## 10. 함정

1. **double dispatch overhead** — 핫 path 에서 메서드 호출 2 회.
2. **Element 추가 시 모든 Visitor 수정** — interface 깨짐. abstract default 메서드로 완화.
3. **stateful visitor 의 race** — traverse 도중 state. snapshot / per-call instance.
4. **return type 다양화** — Java generic `Visitor<R>` 또는 visitor 별 다른 reflect.
5. **재귀 깊이** — deeply nested AST stack overflow. iterative.
6. **함수형 언어에서 visitor 강요** — pattern matching 이 더 자연스러움.

## 11. 관련

- [[../iterator/iterator]] — 구조 traversal.
- [[../composite/composite]] — visitor 의 주 대상.
- [[../strategy/strategy]] — 연산 교체.
