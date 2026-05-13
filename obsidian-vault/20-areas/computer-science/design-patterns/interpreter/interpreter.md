---
title: "Interpreter"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, interpreter, dsl, ast]
---

# Interpreter

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**문법을 표현하는 객체 트리 + 그 트리를 평가하는 interpret() 메서드.** 작은 DSL / 표현식 평가.

## 2. 언제 쓰는가

- **간단한 문법 / DSL** — query / filter / 정책 표현식.
- **rule engine** — business rule 의 DSL.
- **수식 평가** — calculator, spreadsheet formula.
- **regex 같은 작은 언어**.

## 3. 언제 안 쓰는가

- **복잡한 문법** — parser generator (ANTLR / yacc), 또는 기존 언어 embedded.
- **성능 critical** — interpreter 가 느림. JIT / native compile 검토.

## 4. 구조

```
AbstractExpression
  + interpret(context)
  ▲
  ├── TerminalExpression       ← Number, Variable
  │   + interpret() { /* 값 */ }
  │
  └── NonTerminalExpression    ← Add, Multiply, And, Or
      - left, right: Expression
      + interpret() { /* left.interpret() OP right.interpret() */ }
```

## 5. 가장 흔한 예 — 수식 인터프리터

문법:
```
expr ::= number | expr + expr | expr * expr
```

## 6. Java

```java
interface Expression {
    int interpret(Map<String, Integer> ctx);
}

class NumberExpr implements Expression {
    private final int value;
    public NumberExpr(int v) { this.value = v; }
    public int interpret(Map<String, Integer> ctx) { return value; }
}

class VarExpr implements Expression {
    private final String name;
    public VarExpr(String n) { this.name = n; }
    public int interpret(Map<String, Integer> ctx) { return ctx.get(name); }
}

class AddExpr implements Expression {
    private final Expression left, right;
    public AddExpr(Expression l, Expression r) { this.left = l; this.right = r; }
    public int interpret(Map<String, Integer> ctx) {
        return left.interpret(ctx) + right.interpret(ctx);
    }
}

class MulExpr implements Expression {
    private final Expression left, right;
    public MulExpr(Expression l, Expression r) { this.left = l; this.right = r; }
    public int interpret(Map<String, Integer> ctx) {
        return left.interpret(ctx) * right.interpret(ctx);
    }
}

// 사용 — (x + 5) * 2
Expression e = new MulExpr(
    new AddExpr(new VarExpr("x"), new NumberExpr(5)),
    new NumberExpr(2)
);
int result = e.interpret(Map.of("x", 10));    // 30
```

**Spring Expression Language (SpEL)** — interpreter 의 산업 표준:
```java
ExpressionParser parser = new SpelExpressionParser();
Expression exp = parser.parseExpression("user.name.toUpperCase()");
String result = exp.getValue(context, String.class);
```

**MVEL / OGNL / FreeMarker** = 다른 Java DSL interpreter.

**Drools rule engine** = production rule interpreter.

## 7. Python / Django

```python
from abc import ABC, abstractmethod

class Expr(ABC):
    @abstractmethod
    def interpret(self, ctx): ...

class Number(Expr):
    def __init__(self, v): self.v = v
    def interpret(self, ctx): return self.v

class Var(Expr):
    def __init__(self, name): self.name = name
    def interpret(self, ctx): return ctx[self.name]

class Add(Expr):
    def __init__(self, l, r): self.left, self.right = l, r
    def interpret(self, ctx): return self.left.interpret(ctx) + self.right.interpret(ctx)

class Mul(Expr):
    def __init__(self, l, r): self.left, self.right = l, r
    def interpret(self, ctx): return self.left.interpret(ctx) * self.right.interpret(ctx)

# 사용
e = Mul(Add(Var("x"), Number(5)), Number(2))
print(e.interpret({"x": 10}))   # 30
```

**Django Q objects** = 일종의 interpreter:
```python
from django.db.models import Q

filter = Q(name="alice") | (Q(age__gte=18) & Q(active=True))
User.objects.filter(filter)
# Q tree 가 SQL WHERE 로 interpret
```

**Django Template language** — `{{ user.name|upper }}` 가 template interpreter.

**Jinja2 / Mako** — Python template 의 interpreter.

**Python `eval()` / `ast.literal_eval()`** — Python 자체가 interpreter. 사용자 입력에는 절대 X (보안).

## 8. TypeScript / NestJS

```ts
interface Expr {
  interpret(ctx: Record<string, number>): number;
}

class NumberExpr implements Expr {
  constructor(private value: number) {}
  interpret() { return this.value; }
}

class VarExpr implements Expr {
  constructor(private name: string) {}
  interpret(ctx: Record<string, number>) { return ctx[this.name]; }
}

class AddExpr implements Expr {
  constructor(private left: Expr, private right: Expr) {}
  interpret(ctx: Record<string, number>) {
    return this.left.interpret(ctx) + this.right.interpret(ctx);
  }
}

// 사용
const e = new AddExpr(new VarExpr('x'), new NumberExpr(5));
e.interpret({ x: 10 });  // 15
```

**Prisma `where` clause** — JSON DSL 이 SQL 로 interpret:
```ts
prisma.user.findMany({
  where: {
    OR: [
      { name: 'alice' },
      { AND: [{ age: { gte: 18 } }, { active: true }] },
    ],
  },
});
```

**SQL ORM (TypeORM, Sequelize)** — TS object 가 SQL 로 interpret.

**GraphQL resolver** — query AST → field resolver 호출 = interpreter.

**JSON Logic / Liquid / Handlebars** — 사용자 정의 DSL interpreter.

## 9. Go

```go
type Expr interface {
    Interpret(ctx map[string]int) int
}

type Number struct{ V int }
func (n Number) Interpret(ctx map[string]int) int { return n.V }

type Var struct{ Name string }
func (v Var) Interpret(ctx map[string]int) int { return ctx[v.Name] }

type Add struct{ L, R Expr }
func (a Add) Interpret(ctx map[string]int) int {
    return a.L.Interpret(ctx) + a.R.Interpret(ctx)
}

type Mul struct{ L, R Expr }
func (m Mul) Interpret(ctx map[string]int) int {
    return m.L.Interpret(ctx) * m.R.Interpret(ctx)
}

// 사용
e := Mul{L: Add{L: Var{"x"}, R: Number{5}}, R: Number{2}}
result := e.Interpret(map[string]int{"x": 10})   // 30
```

**Go text/template** = 작은 template interpreter.
**Go html/template** — XSS-safe template interpreter.

**OPA Rego** (Open Policy Agent) — Go 로 작성된 policy DSL interpreter, k8s admission control 의 산업 표준.

## 10. Parser + Interpreter 분리

실제 시스템에서:
```
사용자 입력 string
       ↓
   Lexer (token)
       ↓
   Parser (AST)
       ↓
   Interpreter (evaluate)
또는 Compiler (bytecode / machine code)
```

- GoF Interpreter 패턴은 마지막 Interpreter 단.
- 보통 Parser 는 별도 library / generator (ANTLR / yacc / Pratt parser).

## 11. JIT / Compilation vs Interpretation

- **Pure interpreter**: 매 호출마다 AST traverse. 느림.
- **Bytecode interpreter**: AST → bytecode 1 회, 그 후 bytecode 평가. 빠름 (Python, Ruby).
- **JIT**: hot path 를 native code 로 변환. V8, HotSpot, LuaJIT.

GoF Interpreter 는 첫째.

## 12. 함정

1. **복잡한 문법 강행** — `if/else/while/function/closure/...` 까지 가면 visitor + state 결합 + scope 관리 폭주. ANTLR / 기존 언어 embed.
2. **성능 부족** — 매 호출 traversal. 큰 workload 면 bytecode 또는 compile.
3. **error reporting 부재** — line / column 정보 보존 필수 (사용자 친화).
4. **eval() 보안** — 사용자 입력 직접 평가 = RCE. sandbox / 화이트리스트.
5. **Spel / OGNL injection** — 사용자 입력이 expression 으로 들어가면 Equifax breach 류 사고.
6. **scope / variable shadowing** — DSL 의 변수 scope 가 모호하면 디버깅 지옥.

## 13. 관련

- [[../visitor/visitor]] — AST traversal 의 다른 방식.
- [[../composite/composite]] — Expression tree.
- [[../iterator/iterator]] — AST 의 traversal.
- [[../command/command]] — expression 객체화 (interpret() = execute()).
