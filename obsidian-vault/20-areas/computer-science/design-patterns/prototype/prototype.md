---
title: "Prototype"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, prototype, clone]
---

# Prototype

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Creational

## 1. 한 줄

**기존 인스턴스를 복제 (clone) 해서 새 인스턴스 만들기.** 생성 비용이 크거나, 사전 설정된 객체를 base 로 변형.

## 2. 언제 쓰는가

- **객체 생성 비용 ↑** — DB 조회 / 네트워크 / 큰 초기화. clone 이 더 싸다.
- **상속이 부적합한 동적 형태** — JavaScript 의 `Object.create`.
- **사전 설정된 template + 변형** — game 의 적 prefab, document template.
- **undo / snapshot** — 현 상태 clone 후 변경.

## 3. Shallow vs Deep Copy

- **Shallow**: 한 layer 만 복제. 하위 참조는 공유.
- **Deep**: 모든 layer 재귀 복제. 비싸지만 완전 격리.

## 4. 구조

```
Prototype                  ConcretePrototype
  + clone(): Prototype      + clone(): self 의 복사
```

## 5. Java / Spring

```java
public class Document implements Cloneable {
    private String title;
    private List<String> sections = new ArrayList<>();

    @Override
    public Document clone() {
        try {
            Document d = (Document) super.clone();
            d.sections = new ArrayList<>(this.sections);  // deep copy
            return d;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError(e);
        }
    }
}
```

- Java `Cloneable` 은 marker interface — 사실상 broken design.
- **Copy constructor** 가 권장:
  ```java
  public Document(Document other) {
      this.title = other.title;
      this.sections = new ArrayList<>(other.sections);
  }
  ```

**Spring `@Scope("prototype")`**: bean 의 prototype scope 는 매번 새 인스턴스. `@Component @Scope("prototype")` — Spring 이 매 inject 마다 새 객체.

**Lombok**: `@With(toBuilder = true)` 로 immutable 복사 + 일부 필드 수정.

## 6. Python

```python
import copy

class Document:
    def __init__(self):
        self.title = ""
        self.sections = []

doc = Document()
doc.title = "Original"
doc.sections.append("intro")

# shallow
shallow = copy.copy(doc)
shallow.sections.append("added")  # 원본도 영향 (list 공유)

# deep
deep = copy.deepcopy(doc)
deep.sections.append("added")     # 원본 안 영향
```

- 표준 라이브러리 `copy.copy` / `copy.deepcopy` 가 prototype 의 기본.
- `__copy__` / `__deepcopy__` override 로 customize.

```python
class Document:
    def __init__(self):
        self.cache = LargeCache()  # 복사하지 않음 (share)
        self.data = []

    def __deepcopy__(self, memo):
        new = Document()
        new.cache = self.cache       # 공유
        new.data = copy.deepcopy(self.data, memo)
        return new
```

**Django**: `Model.objects.create(pk=None, **instance.__dict__)` 또는 instance 의 `.pk = None; .save()` 패턴.

## 7. TypeScript / Node / React

```ts
class Document {
  title = '';
  sections: string[] = [];

  clone(): Document {
    const d = new Document();
    d.title = this.title;
    d.sections = [...this.sections];   // spread = shallow
    return d;
  }
}

// 또는 structuredClone (Node 17+, browsers)
const cloned = structuredClone(doc);   // deep clone, 표준
```

- ES2022 의 `structuredClone` = deep clone 의 표준.
- spread / `Object.assign` = shallow.
- **Lodash `_.cloneDeep`** = 옛 deep clone 표준.

**React** — state 가 immutable, 복제로 update:
```ts
setUser({ ...user, name: 'new' });
setItems([...items, newItem]);
// Immer 로 더 편하게
setState(produce(state, draft => { draft.user.name = 'new' }));
```

**Prototype-based 상속**:
```ts
const animal = { eat() { console.log('eating'); } };
const dog = Object.create(animal);
dog.bark = function() { console.log('woof'); };
// dog 은 animal 을 prototype 으로 → eat() 도 가능
```

JavaScript 의 prototype chain 자체가 GoF Prototype 패턴.

## 8. Go

```go
type Document struct {
    Title    string
    Sections []string
}

func (d *Document) Clone() *Document {
    sections := make([]string, len(d.Sections))
    copy(sections, d.Sections)
    return &Document{
        Title:    d.Title,
        Sections: sections,
    }
}
```

- Go 는 explicit. `copy()` builtin 으로 slice 복사.
- Map / pointer 는 명시적 복제 필요.
- **JSON marshal/unmarshal trick**:
  ```go
  b, _ := json.Marshal(d)
  var cloned Document
  json.Unmarshal(b, &cloned)
  ```
  — 게으른 deep clone. 비효율 but 안전.

## 9. 함정

1. **Shallow vs deep 혼동** — `clone()` 가 어느 layer 까지 복제하는지 명시.
2. **순환 참조 시 deep copy** — Python `deepcopy` 의 memo 인자, Go 의 visited set.
3. **`Cloneable` (Java) 의 broken design** — `Object.clone()` 이 protected + Exception throw. Copy constructor 권장.
4. **immutable 객체 clone** — 의미 없음. 그냥 reference share.
5. **prototype 의 `__init__` 우회** — Python `copy` 가 `__init__` 호출 안 함. 부작용.
6. **mutable default arguments** — Python `def f(items=[])` 의 list 가 모든 호출 공유. prototype 의 함정과 비슷.

## 10. 관련

- [[../factory-method/factory-method]] — 생성 방식 추상화.
- [[../abstract-factory/abstract-factory]] — 가족 생성.
- [[../singleton/singleton]] — 반대 (인스턴스 1 개).
- [[../memento/memento]] — 상태 snapshot.
