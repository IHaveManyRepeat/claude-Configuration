# Python 最佳实践

## 项目结构

### 标准项目布局

```
project/
├── pyproject.toml        # 项目配置（推荐）
├── src/
│   └── package_name/
│       ├── __init__.py
│       ├── py.typed      # PEP 561 类型标记
│       ├── models.py
│       ├── services.py
│       └── utils.py
├── tests/
│   ├── conftest.py
│   ├── test_models.py
│   └── test_services.py
└── README.md
```

### 使用 `pyproject.toml` 管理项目

```toml
[project]
name = "my-package"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "ruff>=0.4",
    "mypy>=1.10",
]
```

## 类型注解（Type Hints）

### 全面使用类型注解

```python
from typing import Optional, Union

# ✅ 参数和返回值都加类型
def greet(name: str, times: int = 1) -> str:
    return f"Hello, {name}! " * times

# ✅ 使用 modern Python 类型语法（3.10+）
def process(data: str | bytes, encoding: str = "utf-8") -> str:
    if isinstance(data, bytes):
        return data.decode(encoding)
    return data

# ✅ Optional 的正确用法
def find_user(user_id: int) -> User | None:
    ...

# ✅ 泛型
from typing import TypeVar, Generic

T = TypeVar("T")

class Stack(Generic[T]):
    def __init__(self) -> None:
        self._items: list[T] = []

    def push(self, item: T) -> None:
        self._items.append(item)

    def pop(self) -> T:
        return self._items.pop()
```

### 使用 `dataclass` 或 `Pydantic` 定义数据模型

```python
from dataclasses import dataclass, field

# ✅ dataclass 用于内部数据结构
@dataclass(frozen=True)  # frozen=True 不可变
class Point:
    x: float
    y: float

    def distance_to(self, other: "Point") -> float:
        return ((self.x - other.x) ** 2 + (self.y - other.y) ** 2) ** 0.5

# ✅ Pydantic 用于 API 边界数据校验
from pydantic import BaseModel, field_validator

class UserCreate(BaseModel):
    name: str
    email: str
    age: int

    @field_validator("email")
    @classmethod
    def validate_email(cls, v: str) -> str:
        if "@" not in v:
            raise ValueError("Invalid email")
        return v.lower()
```

### 使用 `Protocol` 定义接口

```python
from typing import Protocol

# ✅ 结构化子类型（鸭子类型的类型安全版）
class Drawable(Protocol):
    def draw(self) -> None: ...
    def resize(self, factor: float) -> None: ...

def render(obj: Drawable) -> None:
    obj.draw()

# 任何实现了 draw() 和 resize() 的类都自动满足 Drawable
```

## 函数设计

### 纯函数优先

```python
# ❌ Bad - 修改外部状态
_total = 0
def add_to_total(value: int) -> None:
    global _total
    _total += value

# ✅ Good - 纯函数
def sum_values(values: list[int]) -> int:
    return sum(values)
```

### 使用不可变数据

```python
from dataclasses import dataclass

# ✅ frozen=True 创建不可变对象
@dataclass(frozen=True)
class Config:
    host: str
    port: int
    debug: bool = False

# ✅ 使用 tuple 替代 list 作为不可变序列
def get_coordinates() -> tuple[float, float]:
    return (39.9042, 116.4074)
```

### 迭代器与生成器

```python
# ✅ 生成器处理大数据集，避免一次性加载到内存
def read_large_file(file_path: str) -> Iterator[str]:
    with open(file_path, "r") as f:
        for line in f:
            yield line.strip()

# ✅ 生成器表达式替代列表推导（惰性求值）
total = sum(item.price for item in items if item.active)

# ✅ 使用 itertools
from itertools import chain, groupby, islice

# 扁平化
flat = list(chain.from_iterable(nested_lists))

# 分批处理
def chunked(iterable: Iterable[T], size: int) -> Iterator[list[T]]:
    it = iter(iterable)
    while chunk := list(islice(it, size)):
        yield chunk
```

## 错误处理

### 自定义异常层级

```python
# ✅ 定义业务异常层级
class AppError(Exception):
    """应用基础异常"""

class NotFoundError(AppError):
    """资源未找到"""

class ValidationError(AppError):
    """数据校验失败"""
    def __init__(self, field: str, message: str) -> None:
        self.field = field
        self.message = message
        super().__init__(f"{field}: {message}")

class AuthError(AppError):
    """认证/授权失败"""
```

### 使用 `Result` 模式或异常

```python
# ✅ 方式1：异常用于意外错误
def divide(a: float, b: float) -> float:
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# ✅ 方式2：Result 模式用于可预期的失败
from dataclasses import dataclass
from typing import TypeVar, Union

T = TypeVar("T")
E = TypeVar("E")

@dataclass
class Ok:
    value: object

@dataclass
class Err:
    error: object

Result = Union[Ok, Err]

def safe_divide(a: float, b: float) -> Result:
    if b == 0:
        return Err(ValueError("Cannot divide by zero"))
    return Ok(a / b)
```

### 使用 `contextlib` 管理资源

```python
from contextlib import contextmanager

@contextmanager
def timer(name: str):
    import time
    start = time.perf_counter()
    try:
        yield
    finally:
        elapsed = time.perf_counter() - start
        print(f"{name}: {elapsed:.3f}s")

# 使用
with timer("database query"):
    db.execute(query)
```

## 上下文管理器

```python
# ✅ 使用 contextmanager 装饰器
from contextlib import contextmanager

@contextmanager
def database_transaction(conn):
    cursor = conn.cursor()
    try:
        yield cursor
        conn.commit()
    except Exception:
        conn.rollback()
        raise
    finally:
        cursor.close()
```

## 并发编程

### `asyncio` 异步 I/O

```python
import asyncio
import aiohttp

async def fetch_json(url: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            response.raise_for_status()
            return await response.json()

# ✅ 并发执行多个请求
async def fetch_all(urls: list[str]) -> list[dict]:
    tasks = [fetch_json(url) for url in urls]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

### 线程安全

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# ✅ I/O 密集型用线程池
with ThreadPoolExecutor(max_workers=10) as executor:
    results = list(executor.map(fetch_url, urls))

# ✅ CPU 密集型用进程池
with ProcessPoolExecutor() as executor:
    results = list(executor.map(heavy_computation, data_chunks))
```

## 代码质量工具

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "UP", "B", "SIM", "TCH"]

[tool.mypy]
strict = true
warn_return_any = true
disallow_untyped_defs = true

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
```

## 性能建议

- 使用 `@functools.lru_cache` 缓存纯函数结果
- 列表推导优于 `for` 循环 + `append`
- `str.join()` 优于循环 `+=` 拼接字符串
- 使用 `collections.Counter`/`defaultdict` 替代手动计数/分组
- 热点路径使用 `__slots__` 减少内存占用
- 使用 `pathlib.Path` 替代 `os.path`

## 参考资源

- [Python 官方文档](https://docs.python.org/3/)
- [PEP 索引](https://peps.python.org/)
- [Real Python](https://realpython.com/)
- [Python Patterns](https://github.com/faif/python-patterns)
