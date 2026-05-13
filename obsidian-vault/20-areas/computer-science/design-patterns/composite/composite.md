---
title: "Composite"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, structural, composite, tree]
---

# Composite

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Structural

## 1. 한 줄

**부분-전체 트리를 통일된 interface 로.** Leaf 와 Composite (가지) 가 같은 type 으로 처리.

## 2. 언제 쓰는가

- **파일 시스템** — File (leaf) + Directory (composite).
- **UI 위젯 트리** — Button (leaf) + Panel (children container).
- **수식 / AST** — Number + BinaryOp(left, right).
- **조직도** — Employee + Manager(reports).
- **메뉴 / 카테고리** — 재귀 트리.

## 3. 구조

```
Component (interface)        ← 공통 인터페이스
   + operation()
   ▲
   │
   ├── Leaf                  ← 자식 없음
   │   + operation()
   │
   └── Composite             ← 자식 보유
       - children: Component[]
       + add(c)
       + remove(c)
       + operation() { for child: child.operation() }
```

## 4. Java

```java
interface FileSystemNode {
    int size();
    String name();
}

class File implements FileSystemNode {
    private final String name;
    private final int bytes;
    public File(String n, int b) { this.name = n; this.bytes = b; }
    public int size() { return bytes; }
    public String name() { return name; }
}

class Directory implements FileSystemNode {
    private final String name;
    private final List<FileSystemNode> children = new ArrayList<>();
    public Directory(String n) { this.name = n; }
    public void add(FileSystemNode n) { children.add(n); }
    public int size() {
        return children.stream().mapToInt(FileSystemNode::size).sum();
    }
    public String name() { return name; }
}

// 사용
Directory root = new Directory("/");
root.add(new File("a.txt", 100));
Directory sub = new Directory("docs");
sub.add(new File("b.md", 200));
root.add(sub);

root.size();  // 300
```

**Spring 의 ApplicationContext**: child context 가 parent + 자기 bean. Composite.

## 5. Python / Django

```python
from typing import List

class FileSystemNode:
    def size(self) -> int: ...
    def name(self) -> str: ...

class File(FileSystemNode):
    def __init__(self, name: str, size: int):
        self._name, self._size = name, size
    def size(self): return self._size
    def name(self): return self._name

class Directory(FileSystemNode):
    def __init__(self, name: str):
        self._name = name
        self.children: List[FileSystemNode] = []
    def add(self, n): self.children.append(n)
    def size(self): return sum(c.size() for c in self.children)
    def name(self): return self._name
```

**Django Template tree**: `Template` → `Node` 트리 (TextNode, VariableNode, BlockNode, IfNode). Composite.

**Django Q objects**: `Q(field1=v) | Q(field2=v) & Q(field3=v)` 가 Composite tree.

## 6. TypeScript / React

```ts
interface UINode {
  render(): string;
}

class TextNode implements UINode {
  constructor(private text: string) {}
  render() { return this.text; }
}

class Box implements UINode {
  private children: UINode[] = [];
  add(node: UINode) { this.children.push(node); }
  render() {
    return `<div>${this.children.map(c => c.render()).join('')}</div>`;
  }
}

const root = new Box();
root.add(new TextNode('hello'));
const inner = new Box();
inner.add(new TextNode('world'));
root.add(inner);
console.log(root.render()); // <div>hello<div>world</div></div>
```

**React tree 자체가 Composite**:
- `ReactElement` = Component + props + children.
- 각 element 가 자식 elements 가질 수 있음.
- `<div>` 와 `<MyComponent>` 가 같은 interface (Element).

```tsx
<div>
  <p>hello</p>
  <button onClick={...}>click</button>
</div>
```

— React 의 모든 JSX 가 Composite tree.

## 7. Go

```go
type Node interface {
    Size() int
    Name() string
}

type File struct {
    name string
    size int
}
func (f File) Size() int { return f.size }
func (f File) Name() string { return f.name }

type Directory struct {
    name     string
    children []Node
}
func (d *Directory) Add(n Node) { d.children = append(d.children, n) }
func (d Directory) Size() int {
    total := 0
    for _, c := range d.children {
        total += c.Size()
    }
    return total
}
func (d Directory) Name() string { return d.name }
```

**Go `html/template`**: template tree (text + actions + ranges) = Composite.

## 8. Composite + Visitor

Composite tree 를 traverse 하면서 다른 연산 적용 = [[../visitor/visitor]] 결합.

```python
class Visitor:
    def visit_file(self, f): ...
    def visit_dir(self, d): ...

class File:
    def accept(self, v): v.visit_file(self)

class Directory:
    def accept(self, v):
        v.visit_dir(self)
        for c in self.children: c.accept(v)

class SizeCalculator(Visitor):
    total = 0
    def visit_file(self, f): self.total += f.size
    def visit_dir(self, d): pass
```

## 9. 함정

1. **Leaf 가 Composite 의 메서드 (add/remove) 강제** — Liskov 위반. composite-only 메서드는 Composite 에만.
2. **너무 깊은 트리** — stack overflow. iterative traversal (BFS stack 명시) 권장.
3. **child 의 parent reference 순환** — GC 가 못 처리. weak reference.
4. **타입 안정성 vs 균질성 trade-off** — File 과 Directory 가 같은 type 이면 호출 측 단순하지만 type system 의 분별 약함.

## 10. 관련

- [[../iterator/iterator]] — Composite 트리 traversal.
- [[../visitor/visitor]] — 트리에 연산 적용.
- [[../decorator/decorator]] — 비슷한 구조 (둘 다 같은 interface), 다른 의도.
