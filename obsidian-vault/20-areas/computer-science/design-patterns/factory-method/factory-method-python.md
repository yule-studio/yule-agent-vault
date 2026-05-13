---
title: "Factory Method — Python / Django / FastAPI"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, factory-method, python, django, fastapi]
---

# Factory Method — Python / Django / FastAPI

**[[factory-method|↑ Factory Method hub]]**

## 1. 기본 — abstract base + factory method

```python
from abc import ABC, abstractmethod

class Doc(ABC):
    @abstractmethod
    def render(self): ...

class PdfDoc(Doc):
    def render(self): print("pdf")

class WordDoc(Doc):
    def render(self): print("word")

class DocCreator:
    @staticmethod
    def create(kind: str) -> Doc:
        if kind == "pdf":  return PdfDoc()
        if kind == "word": return WordDoc()
        raise ValueError(kind)
```

## 2. Class method 로 — pythonic

```python
class User:
    def __init__(self, name): self.name = name

    @classmethod
    def from_email(cls, email):
        return cls(email.split("@")[0])

    @classmethod
    def guest(cls):
        return cls("guest")

u = User.from_email("alice@example.com")
g = User.guest()
```

- `@classmethod` 가 가장 idiomatic factory.
- subclass 가 cls 로 자동 동작.

## 3. Registry 기반 — 분기 폭주 회피

```python
class Parser(ABC):
    registry = {}
    extension = None

    @classmethod
    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        if cls.extension:
            Parser.registry[cls.extension] = cls

    @abstractmethod
    def parse(self, data): ...

class JsonParser(Parser):
    extension = "json"
    def parse(self, data): import json; return json.loads(data)

class YamlParser(Parser):
    extension = "yaml"
    def parse(self, data): import yaml; return yaml.safe_load(data)

# 사용
def make_parser(ext) -> Parser:
    return Parser.registry[ext]()

p = make_parser("json")
```

- 새 parser 추가 = 새 subclass 정의만. factory 코드 수정 0.

## 4. Django Model — `create` factory

```python
class Order(models.Model):
    user = models.ForeignKey(User, on_delete=models.CASCADE)
    total = models.DecimalField(max_digits=10, decimal_places=2)

    @classmethod
    def from_cart(cls, user, cart):
        order = cls.objects.create(user=user, total=cart.total())
        for item in cart.items:
            OrderItem.objects.create(order=order, ...)
        return order

# 사용
order = Order.from_cart(request.user, cart)
```

- `Model.objects.create` 가 Django 의 표준 factory.
- domain-specific `from_xxx` 클래스 메서드로 의도 명확.

## 5. Django form `Meta` — ModelForm factory

```python
class OrderForm(forms.ModelForm):
    class Meta:
        model = Order
        fields = ['total']
```

- ModelForm 의 `Meta` 가 form factory.
- form fields 가 model 에서 자동 생성.

## 6. FastAPI `Depends` factory

```python
from fastapi import FastAPI, Depends

class Repo:
    def __init__(self, db): self.db = db

def get_db(): return Database(...)
def get_repo(db = Depends(get_db)): return Repo(db)

app = FastAPI()

@app.get("/users")
def list_users(repo: Repo = Depends(get_repo)):
    return repo.list()
```

- `Depends` 자체가 factory injection.
- 테스트 시 `app.dependency_overrides[get_repo] = lambda: FakeRepo()`.

## 7. Pydantic `BaseModel` factory methods

```python
from pydantic import BaseModel

class Config(BaseModel):
    debug: bool = False
    log_level: str = "INFO"

    @classmethod
    def from_env(cls):
        return cls(
            debug=os.getenv("DEBUG") == "1",
            log_level=os.getenv("LOG_LEVEL", "INFO"),
        )

    @classmethod
    def from_yaml(cls, path):
        return cls(**yaml.safe_load(open(path)))

cfg = Config.from_env()
```

## 8. 함정

1. **factory 가 거대 if-else** — 신규 type 추가마다 코드 수정. registry / `__init_subclass__` 권장.
2. **`__init_subclass__` 의 import 순서** — subclass 가 import 되지 않으면 registry 안 들어감.
3. **abc.ABCMeta + 다른 metaclass 충돌** — Python metaclass 결합 필요.
4. **Pydantic 의 `__init__` override 불가** — `model_validator` 또는 classmethod 사용.

## 9. 관련

- [[factory-method]]
- [[factory-method-java]]
- [[factory-method-typescript]]
- [[factory-method-go]]
