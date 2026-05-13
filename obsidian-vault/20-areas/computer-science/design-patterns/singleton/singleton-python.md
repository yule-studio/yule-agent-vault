---
title: "Singleton — Python / Django / FastAPI"
kind: knowledge
project: computer-science
agent: engineering-agent/tech-lead
status: current
tags: [design-patterns, gof, creational, singleton, python, django, fastapi]
---

# Singleton — Python / Django / FastAPI

**[[singleton|↑ Singleton hub]]**

## 1. Module 자체가 Singleton — 가장 idiomatic

```python
# logger.py
_instance_config = {"level": "INFO"}

def log(msg):
    print(f"[{_instance_config['level']}] {msg}")

def set_level(level):
    _instance_config["level"] = level
```

```python
# 다른 모듈
import logger
logger.log("hello")
logger.set_level("DEBUG")
```

- Python 이 module 을 첫 import 시 1 번만 실행.
- `sys.modules` 가 캐시.
- **거의 모든 Python Singleton 의 정답**.

## 2. `__new__` 으로 강제

```python
class Logger:
    _instance = None

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def log(self, msg):
        print(msg)

a = Logger()
b = Logger()
assert a is b      # 항상 같은 인스턴스
```

- 단순하지만 thread-safe 아님.
- subclass 가 인스턴스를 망가뜨릴 수 있음.

## 3. Thread-safe Singleton

```python
import threading

class Logger:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        with cls._lock:
            if cls._instance is None:
                cls._instance = super().__new__(cls)
        return cls._instance
```

## 4. Decorator 로

```python
def singleton(cls):
    instances = {}
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]
    return get_instance

@singleton
class Logger:
    def log(self, msg): print(msg)

a = Logger()
b = Logger()
assert a is b
```

## 5. Metaclass 로

```python
class SingletonMeta(type):
    _instances = {}
    _lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        with cls._lock:
            if cls not in cls._instances:
                cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]

class Logger(metaclass=SingletonMeta):
    def log(self, msg): print(msg)
```

- 가장 강력 — `Logger()` 호출 자체가 hook.
- subclass 에 자동 상속.
- 그러나 metaclass 가 다른 메타 (e.g. abc.ABCMeta) 와 충돌 가능.

## 6. `functools.lru_cache` — 가벼운 대안

```python
from functools import lru_cache

@lru_cache(maxsize=1)
def get_db_connection():
    return create_connection(...)

# 항상 같은 connection 반환
conn = get_db_connection()
```

- 사실상 함수의 결과를 cache → singleton 효과.
- thread-safe.
- 가장 pythonic.

## 7. Django — settings 와 model

### settings 가 module singleton

```python
from django.conf import settings
settings.DATABASES["default"]    # 어디서든 같은 dict
```

### Singleton model — DB 에 1 row 만

```python
class SiteConfig(models.Model):
    site_name = models.CharField(max_length=100)
    maintenance_mode = models.BooleanField(default=False)

    def save(self, *args, **kwargs):
        self.pk = 1                          # 항상 PK 1 강제
        super().save(*args, **kwargs)

    @classmethod
    def load(cls):
        obj, _ = cls.objects.get_or_create(pk=1)
        return obj

# 사용
config = SiteConfig.load()
config.maintenance_mode = True
config.save()
```

- DB 에 row 1 개 만 존재.
- admin / app 에서 어디서나 같은 row 참조.

### Django middleware — 인스턴스 1 개

```python
# settings.py
MIDDLEWARE = [
    'myapp.middleware.RequestIdMiddleware',
]

# myapp/middleware.py
class RequestIdMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response       # 한 번만 호출 (Django 가 1 인스턴스)

    def __call__(self, request):
        request.request_id = uuid.uuid4().hex
        return self.get_response(request)
```

- Django 가 middleware class 의 인스턴스 1 개만 생성 (worker process 별).
- 따라서 middleware 안 self 변수 = process 별 singleton.

## 8. FastAPI — Dependency Injection

```python
from fastapi import FastAPI, Depends
from functools import lru_cache

app = FastAPI()

@lru_cache(maxsize=1)
def get_settings():
    return Settings()           # 한 번만 생성

@app.get("/")
def root(settings: Settings = Depends(get_settings)):
    return {"name": settings.app_name}
```

- `lru_cache` + `Depends` = singleton 효과.
- FastAPI 가 Depends 결과를 자동 cache (per-request 또는 longer).

### Lifespan event — startup 1 회

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.db = await create_pool()   # 1 회 생성
    yield
    await app.state.db.close()

app = FastAPI(lifespan=lifespan)

@app.get("/users")
async def users(request: Request):
    return await request.app.state.db.fetch("SELECT * FROM users")
```

- `app.state` 가 process-wide 공유 storage.
- 모든 요청이 같은 db pool 공유.

## 9. SQLAlchemy — Engine 의 Singleton

```python
# database.py
from sqlalchemy import create_engine

engine = create_engine("postgresql://...")    # module-level

# 어디서든
from .database import engine
```

- engine 자체가 connection pool.
- 1 process 에 1 engine 권장.

## 10. 함정 (Python specific)

1. **Multi-process (gunicorn / uvicorn worker)** — process 별 별도 Singleton. 진정한 글로벌이면 Redis / shared memory.
2. **`__init__` 재호출** — `__new__` 가 같은 인스턴스 반환해도 `__init__` 은 매번 실행. `__init__` 안에서 idempotent 보장.
3. **Pickle / multiprocessing** — Singleton 도 pickle 시 새 인스턴스. `__reduce__` override.
4. **Test fixture 의 state leak** — module-level state 가 다음 test 로 leak. `pytest` 의 `monkeypatch.setattr` 로 reset.
5. **Django re-load (autoreload)** — dev server 가 module reload 시 Singleton 새로 초기화.
6. **Metaclass 충돌** — `class A(metaclass=SingletonMeta, ABCMeta):` 가 에러. metaclass 결합 필요.

## 11. 권장 — 현대 Python 답

1. **단순 cache** = `@lru_cache(maxsize=1)`.
2. **설정 / 상수** = module-level constant.
3. **DB connection / engine** = module-level + lazy init.
4. **FastAPI** = `Depends(get_xxx)` + `lru_cache`.
5. **Django** = settings + middleware + Singleton model.
6. **테스트 친화** = factory function + DI.

→ 클래스 기반 Singleton (`__new__` / metaclass) 는 **거의 안 씀**. Java/C++ 패턴을 Python 에 옮긴 anti-idiom.

## 12. 관련

- [[singleton]]
- [[singleton-java]]
- [[singleton-typescript]]
- [[singleton-go]]
