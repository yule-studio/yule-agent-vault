---
title: "Memento"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, memento, snapshot, undo]
---

# Memento

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**객체의 상태를 캡슐화 없이 외부 노출해서 저장 / 복원.** Undo / Snapshot 의 기반.

## 2. 언제 쓰는가

- **Undo / Redo** — 텍스트 에디터, Photoshop history.
- **체크포인트** — 게임 save, 트랜잭션 savepoint.
- **시간 여행 디버깅** — Redux DevTools time-travel.
- **테스트 시 fixture 복원**.

## 3. 구조

```
   Originator              Memento (상태 snapshot)
   - state                 + state
   + save() : Memento      + restore() (Originator 에게만)
   + restore(m)

   Caretaker
   - history: List<Memento>
   + push() / pop()
```

## 4. Memento vs Command

- **Memento**: 상태 자체 저장. 큰 객체면 메모리 부담.
- **Command** + undo: 변경 자체를 저장. 메모리 절약, 그러나 모든 작업이 reversible 해야.

## 5. Java

```java
class Editor {
    private String content = "";

    public void type(String text) { content += text; }
    public String getContent() { return content; }

    // memento class
    public static class Snapshot {
        private final String content;
        private Snapshot(String c) { this.content = c; }
    }

    public Snapshot save() { return new Snapshot(content); }
    public void restore(Snapshot s) { this.content = s.content; }
}

class History {
    private final Deque<Editor.Snapshot> stack = new ArrayDeque<>();

    public void backup(Editor e) { stack.push(e.save()); }
    public void undo(Editor e) {
        if (!stack.isEmpty()) e.restore(stack.pop());
    }
}

// 사용
Editor ed = new Editor();
History h = new History();

ed.type("hello ");
h.backup(ed);
ed.type("world");
h.backup(ed);
ed.type("!");

h.undo(ed);  // "hello world"
h.undo(ed);  // "hello "
```

**Hibernate Envers** = entity 의 history 자동 추적 (audit log) = Memento.

## 6. Python

```python
from typing import List
from copy import deepcopy

class Editor:
    def __init__(self):
        self.content = ""

    def type(self, text): self.content += text

    def save(self):
        return Snapshot(deepcopy(self.content))

    def restore(self, snap):
        self.content = snap.content

class Snapshot:
    def __init__(self, content): self.content = content

class History:
    def __init__(self): self.stack: List[Snapshot] = []
    def backup(self, ed): self.stack.append(ed.save())
    def undo(self, ed):
        if self.stack: ed.restore(self.stack.pop())
```

**Django `django-simple-history`** package = model 의 모든 변경 자동 audit (Memento).

**Django Form 의 initial / cleaned_data** = 일종의 snapshot.

## 7. TypeScript / React

```ts
interface Snapshot {
  content: string;
}

class Editor {
  content = '';

  type(text: string) { this.content += text; }

  save(): Snapshot { return { content: this.content }; }
  restore(s: Snapshot) { this.content = s.content; }
}

class History {
  private stack: Snapshot[] = [];
  backup(e: Editor) { this.stack.push(e.save()); }
  undo(e: Editor) {
    const s = this.stack.pop();
    if (s) e.restore(s);
  }
}
```

**Redux Time-Travel Debugging**:
```ts
// Redux DevTools 가 매 action 마다 state snapshot 저장
// step backwards / forwards 가능
```

- 모든 state 가 immutable + history stack = Memento 의 본질.

**React useState + useReducer + history**:
```tsx
function useHistoryState<T>(initial: T) {
  const [stack, setStack] = useState([initial]);
  const [idx, setIdx] = useState(0);

  return {
    state: stack[idx],
    push: (newState: T) => {
      setStack([...stack.slice(0, idx + 1), newState]);
      setIdx(idx + 1);
    },
    undo: () => setIdx(Math.max(0, idx - 1)),
    redo: () => setIdx(Math.min(stack.length - 1, idx + 1)),
  };
}
```

**Immer + history**: state 변경마다 immutable snapshot 자동.

## 8. Go

```go
type EditorSnapshot struct {
    Content string
}

type Editor struct {
    Content string
}

func (e *Editor) Type(text string) { e.Content += text }

func (e *Editor) Save() EditorSnapshot {
    return EditorSnapshot{Content: e.Content}
}

func (e *Editor) Restore(s EditorSnapshot) {
    e.Content = s.Content
}

type History struct{ stack []EditorSnapshot }

func (h *History) Backup(e *Editor) { h.stack = append(h.stack, e.Save()) }
func (h *History) Undo(e *Editor) {
    n := len(h.stack)
    if n == 0 { return }
    e.Restore(h.stack[n-1])
    h.stack = h.stack[:n-1]
}
```

**Database transaction savepoint** = Memento at DB level:
```go
tx, _ := db.Begin()
tx.Exec("...")
tx.Exec("SAVEPOINT a")          // memento
tx.Exec("...")
tx.Exec("ROLLBACK TO SAVEPOINT a")  // restore
tx.Commit()
```

## 9. Memento + Persistent Data Structures (FP)

함수형 언어에서는 **immutable + structural sharing** 으로 Memento 의 비용 절감:

```
state v1 = {a: 1, b: {x: 10}}
state v2 = update v1 (a -> 2) = {a: 2, b: <shared with v1>}
state v3 = update v2 (b.x -> 20) = {a: 2, b: {x: 20}}
```

- v1, v2, v3 모두 share 가능 한 부분 share.
- Clojure / Scala / Immer.js 가 같은 사상.

## 10. 함정

1. **메모리 폭주** — 큰 객체의 every state snapshot. compression / max history.
2. **deep copy 시 순환 참조** — Python `deepcopy` 의 memo 인자, Go visited set.
3. **mutable share** — snapshot 의 nested reference 가 원본과 같으면 둘 다 corrupted. immutable / deep copy.
4. **External 만 보면 restore 불가** — Memento 가 private state 의 모든 것 포함해야. encapsulation 깸.
5. **non-serializable state** — file handle / DB connection 등 비직렬화 state. transient 마킹.
6. **side effect 있는 restore** — 단순 state 복원 + 외부 system 영향 (이미 보낸 email 못 되돌림).

## 11. 관련

- [[../command/command]] — undo 의 다른 방식 (변경 자체 저장).
- [[../prototype/prototype]] — 복제.
- [[../state/state]] — 상태 객체.
