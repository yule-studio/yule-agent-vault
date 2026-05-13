---
title: "Command"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, behavioral, command, undo, queue]
---

# Command

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Behavioral

## 1. 한 줄

**요청을 객체로 만든다.** caller 와 receiver 를 분리. undo / redo / queue / log / macro 가능.

## 2. 언제 쓰는가

- **Undo / Redo** — 텍스트 에디터, Photoshop.
- **트랜잭션 / 매크로** — 여러 command 묶음.
- **Task queue** — Celery / Bull / Sidekiq.
- **이벤트 sourcing** — domain command 가 event 생성.
- **GUI action** — menu / toolbar 가 같은 action 호출.

## 3. 구조

```
Invoker → Command (interface)
              + execute()
              + undo()
              ▲
              │
         ConcreteCommand
              - receiver: Receiver
              + execute() { receiver.action() }
              + undo() { receiver.reverse() }
```

## 4. Java / Spring

```java
interface Command {
    void execute();
    void undo();
}

class Light {
    void on() { System.out.println("on"); }
    void off() { System.out.println("off"); }
}

class LightOn implements Command {
    private Light light;
    public LightOn(Light l) { this.light = l; }
    public void execute() { light.on(); }
    public void undo() { light.off(); }
}

class Remote {
    private List<Command> history = new ArrayList<>();
    void press(Command c) {
        c.execute();
        history.add(c);
    }
    void undoLast() {
        if (!history.isEmpty()) {
            history.remove(history.size()-1).undo();
        }
    }
}
```

**Spring**:
- **`@EventListener` + `ApplicationEvent`** — domain event 가 command 형태.
- **Spring Batch `Tasklet`** — 한 step 의 command.
- **Spring Integration `Gateway`** — message 가 command 객체.

## 5. Python / Django

```python
from abc import ABC, abstractmethod
from typing import List

class Command(ABC):
    @abstractmethod
    def execute(self): ...
    @abstractmethod
    def undo(self): ...

class Light:
    def on(self): print("on")
    def off(self): print("off")

class LightOn(Command):
    def __init__(self, light): self.light = light
    def execute(self): self.light.on()
    def undo(self): self.light.off()

class Remote:
    def __init__(self): self.history: List[Command] = []
    def press(self, cmd):
        cmd.execute()
        self.history.append(cmd)
    def undo(self):
        if self.history:
            self.history.pop().undo()
```

**Django management command**:
```python
# myapp/management/commands/sync_data.py
from django.core.management.base import BaseCommand

class Command(BaseCommand):
    help = "Sync data from external API"

    def add_arguments(self, parser):
        parser.add_argument('--dry-run', action='store_true')

    def handle(self, *args, **options):
        # command logic
        ...
```

`python manage.py sync_data --dry-run` 으로 실행 — Django 의 CLI 가 Command 패턴.

**Celery task** = Command:
```python
@app.task
def send_email(to, subject, body):
    ...

send_email.delay("alice@example.com", "Hi", "Hello")  # queue 에 enqueue
```

## 6. TypeScript / NestJS / React

```ts
interface Command {
  execute(): void;
  undo(): void;
}

class Light {
  on() { console.log('on'); }
  off() { console.log('off'); }
}

class LightOn implements Command {
  constructor(private light: Light) {}
  execute() { this.light.on(); }
  undo() { this.light.off(); }
}

class Remote {
  private history: Command[] = [];
  press(cmd: Command) {
    cmd.execute();
    this.history.push(cmd);
  }
  undo() {
    this.history.pop()?.undo();
  }
}
```

**NestJS CQRS module** = Command 패턴:
```ts
class CreateUserCommand {
  constructor(public readonly email: string, public readonly password: string) {}
}

@CommandHandler(CreateUserCommand)
export class CreateUserHandler {
  async execute(cmd: CreateUserCommand) {
    // create user
  }
}

// 사용
await this.commandBus.execute(new CreateUserCommand(email, password));
```

**Redux Action** = Command (with reducer 가 receiver):
```ts
dispatch({ type: 'ADD_TODO', payload: { text: 'buy milk' } });
```

**Bull / BullMQ Job queue** = Command queue:
```ts
await queue.add('sendEmail', { to: 'alice', body: 'hi' });
```

## 7. Go

```go
type Command interface {
    Execute() error
    Undo() error
}

type Light struct{}
func (l *Light) On()  { fmt.Println("on")  }
func (l *Light) Off() { fmt.Println("off") }

type LightOn struct{ light *Light }
func (c *LightOn) Execute() error { c.light.On();  return nil }
func (c *LightOn) Undo()    error { c.light.Off(); return nil }

type Remote struct{ history []Command }
func (r *Remote) Press(c Command) error {
    if err := c.Execute(); err != nil { return err }
    r.history = append(r.history, c)
    return nil
}
func (r *Remote) Undo() error {
    n := len(r.history)
    if n == 0 { return nil }
    cmd := r.history[n-1]
    r.history = r.history[:n-1]
    return cmd.Undo()
}
```

**Go cobra CLI library** = Command 패턴:
```go
var rootCmd = &cobra.Command{
    Use:   "myapp",
    Short: "...",
    Run: func(cmd *cobra.Command, args []string) {
        // command logic
    },
}
```

**Kubernetes Operator** 의 reconciler 가 본질적으로 Command receiver.

## 8. Event Sourcing 과 결합

```
User clicks "AddToCart"
  ↓
AddToCartCommand created
  ↓
CommandHandler.execute(cmd)
  ↓
Domain validates
  ↓
ItemAddedToCart Event emitted
  ↓
Event stored + projection updated
```

- Command = "지시" (validation 거침).
- Event = "이미 일어난 사실" (immutable).

## 9. Macro / Composite Command

```python
class MacroCommand(Command):
    def __init__(self, commands):
        self.commands = commands
    def execute(self):
        for c in self.commands: c.execute()
    def undo(self):
        for c in reversed(self.commands): c.undo()

macro = MacroCommand([cmd1, cmd2, cmd3])
remote.press(macro)
```

## 10. 함정

1. **undo 의 부분 실패** — execute 가 성공한 후 undo 가 실패하면 상태 불일치. compensation pattern.
2. **idempotent command** — queue retry 시 같은 command 두 번 실행. idempotency key.
3. **command 직렬화** — distributed queue 에 보낼 때 receiver / context 직렬화 어려움.
4. **command history 의 메모리 / 디스크** — undo stack 이 무한하면 메모리 누수. 한도 둠.
5. **Macro command 의 transaction** — 일부 command 실패 시 전체 rollback or partial commit?
6. **stateful receiver** — command 가 모르는 receiver state 의존 시 race.

## 11. 관련

- [[../memento/memento]] — 상태 snapshot (undo 의 다른 방식).
- [[../chain-of-responsibility/chain-of-responsibility]] — command 의 처리 chain.
- [[../strategy/strategy]] — 다른 의도 (알고리즘 교체).
