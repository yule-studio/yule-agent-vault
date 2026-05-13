---
title: "Abstract Factory"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, abstract-factory]
---

# Abstract Factory

**[[../design-patterns|↑ 디자인 패턴]]** · GoF Creational

## 1. 한 줄

**관련 객체 가족을 생성하는 인터페이스.** 같은 family 의 객체들이 서로 호환되도록.

## 2. 의도

- "이 family 의 모든 객체를 한 묶음으로" 생성.
- 가족 간 호환성 보장 (예: Windows UI 위젯 vs Mac UI 위젯 섞이지 않음).
- 새 family 추가가 쉬움. 새 product 추가는 모든 factory 수정 (대가).

## 3. 언제 쓰는가

- **plat-formspecific UI** — Win / Mac / Linux 의 button + checkbox + dialog 가족.
- **DB driver family** — PostgreSQL 의 connection + statement + result. MySQL 의 같은 family.
- **Theme system** — dark / light 의 button / input / card family.
- **cross-cutting test fixture** — prod / test family.

## 4. Factory Method vs Abstract Factory

- **Factory Method**: 한 객체 (subclass 가 어떤 객체) 생성.
- **Abstract Factory**: 관련 객체 가족 (Window + Button + Checkbox).

## 5. 구조

```
AbstractFactory                   AbstractProductA  AbstractProductB
  + createA(): ProductA           Button             Checkbox
  + createB(): ProductB           ▲                  ▲
  ▲                              │                  │
  │                       WinButton MacButton  WinCheckbox MacCheckbox
WinFactory             ─►
MacFactory             ─►
```

## 6. Java / Spring

```java
interface Button { void render(); }
interface Checkbox { void check(); }

interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

class WinFactory implements UIFactory {
    public Button createButton() { return new WinButton(); }
    public Checkbox createCheckbox() { return new WinCheckbox(); }
}

class MacFactory implements UIFactory {
    public Button createButton() { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

// 사용
UIFactory f = isWindows ? new WinFactory() : new MacFactory();
f.createButton().render();
f.createCheckbox().check();
```

**Spring 적용**: `@Profile("windows")` `@Component class WinFactory implements UIFactory {...}` — 프로필 별 다른 factory bean 자동 주입.

## 7. Python / Django / FastAPI

```python
class UIFactory(Protocol):
    def create_button(self) -> Button: ...
    def create_checkbox(self) -> Checkbox: ...

class WinFactory:
    def create_button(self): return WinButton()
    def create_checkbox(self): return WinCheckbox()

class MacFactory:
    def create_button(self): return MacButton()
    def create_checkbox(self): return MacCheckbox()

# Django settings + custom
import platform
factory = WinFactory() if platform.system() == "Windows" else MacFactory()
```

**Django 적용**: `STORAGES` 설정이 abstract factory 패턴 — file / staticfile / media 의 storage backend family.

**FastAPI 적용**: `Depends` 로 환경별 factory 주입.

## 8. TypeScript / NestJS / React

```ts
interface UIFactory {
  createButton(): Button;
  createCheckbox(): Checkbox;
}

class MaterialFactory implements UIFactory {
  createButton() { return new MaterialButton(); }
  createCheckbox() { return new MaterialCheckbox(); }
}

class TailwindFactory implements UIFactory {
  createButton() { return new TailwindButton(); }
  createCheckbox() { return new TailwindCheckbox(); }
}
```

**NestJS 적용**:
```ts
@Module({
  providers: [
    {
      provide: 'UI_FACTORY',
      useFactory: () => isDark() ? new DarkFactory() : new LightFactory(),
    },
  ],
})
```

**React 적용**: Theme context 가 본질적으로 Abstract Factory — `useTheme().Button`, `useTheme().Card` 가 같은 family 제공.

## 9. Go

```go
type UIFactory interface {
    NewButton() Button
    NewCheckbox() Checkbox
}

type WinFactory struct{}
func (WinFactory) NewButton() Button { return &WinButton{} }
func (WinFactory) NewCheckbox() Checkbox { return &WinCheckbox{} }

type MacFactory struct{}
func (MacFactory) NewButton() Button { return &MacButton{} }
func (MacFactory) NewCheckbox() Checkbox { return &MacCheckbox{} }

// 사용
var f UIFactory = WinFactory{}
b := f.NewButton()
```

`database/sql` 도 일종의 abstract factory — Driver interface 가 Connection / Stmt / Tx family 를 만듦.

## 10. 함정

1. **product 추가가 어려움** — 모든 ConcreteFactory 수정 필요. family stable 한 경우에만.
2. **factory 폭주** — family 수가 많으면 ConcreteFactory N×M 가 됨. Bridge / Strategy 결합.
3. **interface bloat** — family 의 모든 product 가 모든 factory 에 강제됨. 일부 family 가 그 product 없으면?
4. **Spring `@Primary` / `@Qualifier` 혼란** — 여러 factory bean 의 선택.

## 11. 관련

- [[../factory-method/factory-method]] — 단일 객체.
- [[../builder/builder]] — 단일 복잡 객체.
- [[../prototype/prototype]] — 복제.
- [[../bridge/bridge]] — abstraction × implementation 분리.
