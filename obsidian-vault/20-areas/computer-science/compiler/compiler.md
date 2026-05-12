---
title: "컴파일러 / 인터프리터 (Compiler / Interpreter)"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-13T14:00:00+09:00
tags:
  - compiler
  - interpreter
  - lexer
  - parser
  - llvm
  - garbage-collection
---

# 컴파일러 / 인터프리터 (Compiler / Interpreter)

| 문서 버전 | 작성일 | 작성자 | 주요 변경 사항 |
| --- | --- | --- | --- |
| v.1.0.0 | 2026-05-13 | engineering-agent/tech-lead | 사전 수준 |

**[[../computer-science|↑ computer-science]]**

---

## 1. 한 줄 정의

**소스 코드를 다른 형태 (기계어 / IR / 다른 언어) 로 변환**. 어휘 → 구문 → 의미 →
최적화 → 코드 생성의 5 단계가 표준. 모든 프로그래밍 언어의 토대.

---

## 2. 역사

| 연도 | 사건 |
| --- | --- |
| 1957 | FORTRAN 컴파일러 (John Backus, IBM) — 최초 |
| 1959 | COBOL |
| 1972 | C 컴파일러 (Ritchie) |
| 1977 | Dragon Book 1 판 (Aho/Ullman) |
| 1986 | Pascal Compiler (Wirth) |
| 2000 | LLVM (Chris Lattner) |
| 2003 | V8 / SpiderMonkey JIT |
| 2009 | Go |
| 2010 | Rust |
| 2014 | Swift / LLVM 기반 |
| 2024 | MLIR / 멀티 레벨 IR |

---

## 3. 컴파일러 vs 인터프리터

### 3.1 비교

| 기준 | 컴파일러 | 인터프리터 |
| --- | --- | --- |
| 변환 시점 | 미리 (AOT) | 실행 중 |
| 출력 | 기계어 / IR | 결과 |
| 속도 | 빠름 | 느림 |
| 이식성 | 타겟 별 | 인터프리터만 있으면 |
| 디버깅 | 어려움 | 쉬움 |
| 예 | C, Rust, Go | Python (CPython), JS (V8 도 JIT 포함) |

### 3.2 하이브리드

- **JIT** (Just-In-Time) — V8, JVM HotSpot, .NET CLR, PyPy
- **AOT** + **JIT** 결합 — Android ART
- **Tracing JIT** vs **Method JIT**

### 3.3 Transpiler / Source-to-Source

- TypeScript → JavaScript
- Babel
- CoffeeScript / Dart 2 web

---

## 4. 컴파일러 5 단계

```
Source → [Lexer] → Tokens → [Parser] → AST → [Semantic] → Annotated AST
       → [IR/Optimizer] → IR → [CodeGen] → Target
```

### 4.1 어휘 분석 (Lexical Analysis / Scanner)

- 입력 문자열 → 토큰 스트림
- 정규 표현식 / DFA / NFA 기반
- 도구: **lex / flex**

```
"int x = 42;"
→ [INT, ID(x), EQ, NUM(42), SEMI]
```

### 4.2 구문 분석 (Parsing)

- 토큰 → AST (Abstract Syntax Tree)
- **Context-Free Grammar (CFG)**
- **Top-down** — Recursive Descent, LL(k), LL(*)
- **Bottom-up** — LR(0), SLR, LALR, LR(1)
- 도구: **yacc / bison / ANTLR / tree-sitter**

### 4.3 의미 분석 (Semantic Analysis)

- 타입 검사
- 스코프 / 심볼 테이블
- 선언 / 사용 매칭
- 타입 추론 (Hindley-Milner)

### 4.4 중간 표현 (IR) + 최적화

#### IR 종류
- **AST** — 트리
- **3-address code** — `t1 = a + b`
- **SSA** (Static Single Assignment) — 각 변수 한 번만 정의
- **CFG** (Control Flow Graph)

#### 최적화 패스
- Constant folding — `2 + 3 → 5`
- Constant propagation
- Dead code elimination
- Common subexpression elimination
- Loop invariant motion
- Inlining
- Tail call elimination
- Loop unrolling
- Vectorization (SIMD)
- Register allocation (Graph coloring, Linear scan)

### 4.5 코드 생성

- IR → 어셈블리 / 기계어
- ISA 별 (x86-64, ARM, RISC-V)
- Instruction selection
- Instruction scheduling
- Register allocation

---

## 5. LLVM

### 5.1 구조

```
Frontend (Clang / Rust / Swift) → LLVM IR → Optimizer → Backend (x86/ARM/...)
```

- 통합 IR — 한 번 최적화 작성 → 모든 언어 / 타겟 혜택
- Chris Lattner 2003

### 5.2 LLVM IR 예
```llvm
define i32 @add(i32 %a, i32 %b) {
entry:
    %sum = add i32 %a, %b
    ret i32 %sum
}
```

### 5.3 사용 사례
- Clang (C/C++/ObjC)
- Rust
- Swift
- Julia
- Mojo

---

## 6. JIT 컴파일

### 6.1 흐름

```
Bytecode → 실행 (Interpret) → "hot" 감지 → JIT Compile → 기계어 → 실행
```

### 6.2 JIT 의 장점

- 동적 정보 (실제 타입, 분기 빈도) 로 최적화
- Inlining 더 공격적
- Deoptimize 후 재시도 가능

### 6.3 주요 JIT

- **V8 (Ignition + TurboFan)** — JavaScript
- **HotSpot (C1 + C2)** — Java
- **GraalVM** — 다언어, Truffle framework
- **CLR (RyuJIT)** — C#
- **PyPy** — RPython 기반

### 6.4 Tiered Compilation

1. Interpret
2. C1 (간단 JIT)
3. C2 (공격적 JIT, 시간 많이 걸림)

---

## 7. 타입 시스템

### 7.1 정적 vs 동적

| 정적 | 동적 |
| --- | --- |
| 컴파일 시 검사 | 런타임 검사 |
| C, Rust, Java, Go | Python, JS, Ruby |
| 안전성 ↑ | 유연성 ↑ |

### 7.2 강 vs 약 타입

- **Strong** — 암시적 변환 적음 (Python, Haskell)
- **Weak** — 자동 변환 많음 (JS, C)

### 7.3 타입 추론 (Type Inference)

- **Hindley-Milner** (HM) — ML, Haskell, OCaml 의 기반
- 1969 Hindley, 1978 Milner
- 모든 타입을 명시 안 해도 추론
- Rust 도 부분적으로 HM

### 7.4 타입 시스템 분류

- **Nominal** — 이름이 같아야 (Java, C++)
- **Structural** — 구조가 같으면 (TypeScript, OCaml)
- **Subtyping** — 부모 / 자식 (Java)
- **Parametric Polymorphism** — Generic
- **Dependent Types** — 값에 의존 (Idris, Agda, Coq)

---

## 8. 메모리 관리

### 8.1 수동 (Manual)

- C/C++ — `malloc/free`, `new/delete`
- 빠름, 위험 (use-after-free, leak)

### 8.2 RAII (Resource Acquisition Is Initialization)

- C++ 의 소멸자
- Rust 의 Drop
- 스코프 종료 시 자동 해제

### 8.3 Reference Counting

- Python, Swift, Objective-C
- 즉시 해제, 순환 참조 문제 (Weak ref 로 해결)

### 8.4 Garbage Collection (GC)

#### Mark-and-Sweep
- Root 부터 도달 가능한 객체 마킹 → 나머지 회수

#### Generational
- Young / Old 세대 분리
- "Most objects die young"
- Minor GC (young) 자주, Major GC 가끔

#### Tri-color (Concurrent)
- White (미방문), Gray (방문 중), Black (완료)
- Go, Java G1, ZGC

#### Reference Counting + GC
- CPython — RefCount + Cycle Collector

#### Compacting GC
- 단편화 해소, 압축
- Java G1, Shenandoah

#### Pause Time
- **Stop-the-world** — 전체 중단
- **Concurrent** — 응용과 병행
- **ZGC (Java)** — <10ms

---

## 9. 런타임 (Runtime)

### 9.1 구성요소

- 메모리 할당자
- GC
- 스레드 스케줄러 (Go 의 M:N goroutine)
- 표준 라이브러리
- FFI (Foreign Function Interface)

### 9.2 Go 의 GMP 모델

- **G** oroutine — 사용자 레벨 스레드
- **M** achine — OS 스레드
- **P** rocessor — 스케줄링 context
- Work-stealing 스케줄러

### 9.3 Erlang BEAM
- 수만 ~ 수백만 경량 프로세스
- Preemptive 스케줄링
- Hot code reload

---

## 10. 파서 도구

| 도구 | 종류 |
| --- | --- |
| **lex/flex** | Lexer 생성 |
| **yacc/bison** | LALR Parser 생성 |
| **ANTLR** | LL(*) — 자바 / 다언어 |
| **tree-sitter** | Incremental, error-tolerant — GitHub, Neovim |
| **PEG.js / pest** | PEG (Parsing Expression Grammar) |
| **Recursive Descent** | 손코딩 — Rust, Go 컴파일러 |

---

## 11. 실전 — 인터프리터 만들기 (간단)

### 11.1 산술 표현식 인터프리터 (Python)
```python
import re

# Lexer
def tokenize(s):
    tokens = re.findall(r'\d+|[+\-*/()]', s)
    return tokens

# Parser + Evaluator (Recursive Descent)
class Parser:
    def __init__(self, tokens):
        self.tokens = tokens
        self.pos = 0
    
    def peek(self):
        return self.tokens[self.pos] if self.pos < len(self.tokens) else None
    
    def consume(self):
        tok = self.peek(); self.pos += 1; return tok
    
    def expr(self):     # expr = term (('+' | '-') term)*
        v = self.term()
        while self.peek() in ('+', '-'):
            op = self.consume()
            r = self.term()
            v = v + r if op == '+' else v - r
        return v
    
    def term(self):     # term = factor (('*' | '/') factor)*
        v = self.factor()
        while self.peek() in ('*', '/'):
            op = self.consume()
            r = self.factor()
            v = v * r if op == '*' else v / r
        return v
    
    def factor(self):   # factor = NUMBER | '(' expr ')'
        tok = self.consume()
        if tok == '(':
            v = self.expr()
            self.consume()   # ')'
            return v
        return int(tok)

print(Parser(tokenize("2 + 3 * (4 - 1)")).expr())   # 11
```

---

## 12. 함정 / 안티패턴

### 함정 1 — 정규식으로 코드 파싱
HTML / Markdown / 코드는 컨텍스트 프리. 파서 필수.

### 함정 2 — Left Recursion
LL 파서는 좌측 재귀 무한 루프. 우측으로 재작성.

### 함정 3 — `eval` 의 보안 / 성능
JS / Python eval — 인젝션 위험, 최적화 불가.

### 함정 4 — JIT warm-up 무시
벤치마크는 JIT 가 데워진 후 측정.

### 함정 5 — GC 무시
긴 latency / 메모리 사용 측정 안 함.

### 함정 6 — 타입 추론에 의존
복잡한 코드에선 명시적 타입이 가독성 ↑.

### 함정 7 — Hand-rolled parser 의 오버엔지니어링
간단한 DSL 엔 `regex + state machine` 충분.

---

## 13. 면접 / 토픽

1. **컴파일러 5 단계**.
2. **LL vs LR 파서**.
3. **SSA 가 무엇이고 왜**.
4. **JIT vs AOT**.
5. **GC 알고리즘 종류**.
6. **Generational GC 의 가설**.
7. **타입 추론 (Hindley-Milner)**.
8. **LLVM 의 의의**.
9. **인터프리터 vs 컴파일러 vs JIT**.
10. **언어 설계 시 정적 vs 동적 타이핑 선택**.

---

## 14. 학습 자료

- **Dragon Book** — Compilers: Principles, Techniques, and Tools (Aho/Sethi/Ullman/Lam)
- **Engineering a Compiler** (Cooper / Torczon) — 더 현대적
- **Crafting Interpreters** (Bob Nystrom) — 무료 https://craftinginterpreters.com — 강추
- **Modern Compiler Implementation in ML** (Appel)
- **LLVM Tutorial** — Kaleidoscope
- **Garbage Collection Handbook** (Jones / Hosking / Moss)
- **The Implementation of Functional Programming Languages** (Peyton Jones) — 무료

---

## 15. 관련

- [[../computer-architecture/computer-architecture]] — ISA, 어셈블리
- [[../operating-system/operating-system]] — 메모리, 시스템 콜
- [[../data-structure/trees/trees]] — AST
- [[../algorithm/dynamic-programming/dynamic-programming]] — 최적화
- [[../computer-science|↑ computer-science]]
