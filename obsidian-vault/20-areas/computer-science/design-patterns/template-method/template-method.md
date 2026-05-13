---
title: "Template Method"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, template-method, inheritance]
---

# Template Method

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**알고리즘 골격은 상위 클래스, 일부 step 만 자식이 구현.** Inheritance 기반.

## 2. Template Method vs Strategy

| | Template Method | Strategy |
| --- | --- | --- |
| 기법 | 상속 | 합성 |
| 변경 시점 | 컴파일 | 런타임 |
| 골격 변경 | super class 수정 | 알고리즘 자체 교체 |
| 결합 | 강함 | 약함 |

## 3. 언제 쓰는가

- **알고리즘의 step 순서 / 골격 고정** — 일부만 변동.
- **framework 의 hook method** — 자식이 일부 override.
- **개념적 동일 흐름 + 세부 다름** — Report 의 fetch → process → render, 각 step 자식이.

## 4. 구조

```
AbstractClass
  + templateMethod() (final) {
      step1();          ← 자식 override
      step2();          ← 자식 override
      finalize();        ← 공통 / 또는 hook
    }
  # step1() abstract
  # step2() abstract
  ▲
  └── ConcreteClassA   ← step1, step2 구현
  └── ConcreteClassB
```

## 5. Java / Spring

```java
abstract class ReportGenerator {
    // Template method
    public final void generate() {
        Object data = fetchData();
        Object processed = process(data);
        write(processed);
        cleanup();           // hook
    }

    protected abstract Object fetchData();
    protected abstract Object process(Object data);
    protected abstract void write(Object data);

    // hook — 자식이 override 선택적
    protected void cleanup() { /* default no-op */ }
}

class PdfReport extends ReportGenerator {
    protected Object fetchData() { return /* db */; }
    protected Object process(Object d) { return /* pdf 변환 */; }
    protected void write(Object d) { /* pdf 저장 */ }
}

class CsvReport extends ReportGenerator {
    protected Object fetchData() { return /* db */; }
    protected Object process(Object d) { return /* csv 변환 */; }
    protected void write(Object d) { /* csv 저장 */ }
    @Override
    protected void cleanup() { /* csv 별도 cleanup */ }
}
```

**Spring `JdbcTemplate.query`** = Template method:
```java
List<User> users = jdbcTemplate.query(
    "SELECT * FROM users",
    (rs, i) -> new User(rs.getLong("id"), rs.getString("name"))
);
```
- `JdbcTemplate` 이 connection 열기 → statement → resultSet → close 의 골격.
- 사용자는 row mapper 만 제공.

**Spring `RestTemplate` / `WebClient`** 도 유사.

**Spring `AbstractApplicationContext.refresh()`** — context init 의 골격 + 자식 (XML / Annotation / Web) override.

## 6. Python / Django

```python
from abc import ABC, abstractmethod

class ReportGenerator(ABC):
    def generate(self):                    # template method
        data = self.fetch_data()
        processed = self.process(data)
        self.write(processed)
        self.cleanup()

    @abstractmethod
    def fetch_data(self): ...
    @abstractmethod
    def process(self, data): ...
    @abstractmethod
    def write(self, data): ...

    def cleanup(self): pass                # hook

class PdfReport(ReportGenerator):
    def fetch_data(self): return ...
    def process(self, d): return ...
    def write(self, d): pass
```

**Django Class-Based Views** = Template Method 의 산업 표준:
```python
from django.views.generic import ListView

class UserListView(ListView):
    model = User
    template_name = 'users/list.html'
    paginate_by = 20

    def get_queryset(self):           # hook
        return super().get_queryset().filter(active=True)

    def get_context_data(self, **kwargs):  # hook
        ctx = super().get_context_data(**kwargs)
        ctx['extra'] = 'data'
        return ctx
```

- `View.dispatch()` → `get()` → `get_queryset()` → `get_context_data()` → `render_to_response()` 의 골격.
- 자식이 일부 hook 만 override.

**Django REST Framework `APIView`** / **GenericAPIView** 도 같은 패턴.

**Django Form `is_valid()` → `clean()` → `clean_<field>()`** 도 Template.

## 7. TypeScript / NestJS / React

```ts
abstract class ReportGenerator {
  generate() {
    const data = this.fetchData();
    const processed = this.process(data);
    this.write(processed);
    this.cleanup();
  }

  protected abstract fetchData(): unknown;
  protected abstract process(data: unknown): unknown;
  protected abstract write(data: unknown): void;

  protected cleanup(): void {}      // hook
}

class PdfReport extends ReportGenerator {
  protected fetchData() { return null; }
  protected process(d: unknown) { return d; }
  protected write(d: unknown) {}
}
```

**React Class component lifecycle** = Template (deprecated but illustrative):
```tsx
class MyComponent extends React.Component {
  componentDidMount() { /* hook */ }
  componentDidUpdate() { /* hook */ }
  componentWillUnmount() { /* hook */ }
  render() { /* abstract — 모든 component */ }
}
```

**NestJS Pipe / Guard / Interceptor**:
```ts
@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string): number {       // abstract method
    const n = parseInt(value, 10);
    if (isNaN(n)) throw new BadRequestException();
    return n;
  }
}
```

- `PipeTransform.transform` 이 NestJS 가 호출하는 template method.

## 8. Go

Go 는 상속 없음 → 보통 **함수 합성** 또는 **interface + embedding** 으로 비슷한 효과.

```go
type ReportSteps interface {
    FetchData() interface{}
    Process(data interface{}) interface{}
    Write(data interface{})
}

func GenerateReport(s ReportSteps) {
    data := s.FetchData()
    processed := s.Process(data)
    s.Write(processed)
}

// 구현
type PdfReport struct{}
func (p PdfReport) FetchData() interface{} { return nil }
func (p PdfReport) Process(d interface{}) interface{} { return d }
func (p PdfReport) Write(d interface{}) {}

GenerateReport(PdfReport{})
```

또는 함수 자체를 받기:
```go
type Report struct {
    Fetch   func() interface{}
    Process func(data interface{}) interface{}
    Write   func(data interface{})
}

func (r Report) Generate() {
    r.Write(r.Process(r.Fetch()))
}
```

→ Go 는 Strategy 와 Template 의 경계가 사실상 사라짐.

## 9. Hook Method 분류

- **Required** (abstract): 자식이 반드시 구현.
- **Optional** (default impl): 자식이 선택적 override.
- **Before / After hook**: 일부 step 전/후.

```python
class ReportGenerator:
    def generate(self):
        self.before()
        data = self.fetch_data()
        self.after()

    def before(self): pass    # optional hook
    def after(self): pass     # optional hook
    @abstractmethod
    def fetch_data(self): ... # required
```

## 10. 함정

1. **Hollywood Principle ("Don't call us, we'll call you")** — 자식이 super 호출 안 함. framework 가 호출.
2. **자식이 골격 깨뜨림** — `super()` 안 호출 또는 template method override.
3. **deep inheritance** — 3+ 단계 상속이면 합성 / Strategy 재검토.
4. **이름 모호** — `super.method()` 안 호출 시 어떤 step 이 실행 안 됐는지 추적 어려움.
5. **Liskov 위반** — 자식이 부모 contract 깨면 client 가 부모 expecting 인 곳에서 fail.
6. **fragile base class** — 부모 변경이 모든 자식 영향. composition 권장.
7. **테스트 시 mock 어려움** — abstract class 의 일부 method 만 mock 하려면 spy 필요.

## 11. 관련

- [[../strategy/strategy]] — 비슷한 의도, 합성 기반.
- [[../factory-method/factory-method]] — 자식이 객체 생성 결정.
- [[../command/command]] — 행동 객체화.
