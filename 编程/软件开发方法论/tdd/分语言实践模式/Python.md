# TDD 实践模式 — Python

#编程语言/Python #软件开发方法论/TDD

---

## 工具链

| 工具 | 用途 | 安装 |
|------|------|------|
| pytest（推荐） | 测试框架 | `pip install pytest` |
| pytest-cov | 覆盖率 | `pip install pytest-cov` |
| pytest-asyncio | 异步测试 | `pip install pytest-asyncio` |
| pytest-mock | Mock 封装 | `pip install pytest-mock` |

### pytest.ini 或 pyproject.toml

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = -v --tb=short
```

```toml
# pyproject.toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-v --tb=short"
```

---

## 纯函数 TDD 示例

### RED — 写测试

```python
# tests/test_pricing.py
import pytest
from pricing import calculate_discount


class TestCalculateDiscount:
    def test_returns_zero_below_threshold(self):
        assert calculate_discount(50, threshold=100) == 0

    def test_applies_vip_discount(self):
        assert calculate_discount(200, threshold=100, is_vip=True) == 20

    def test_raises_on_negative_amount(self):
        with pytest.raises(ValueError, match="must be positive"):
            calculate_discount(-1, threshold=100)
```

### GREEN — 最简实现

```python
# pricing.py
def calculate_discount(amount: float, threshold: float, is_vip: bool = False) -> int:
    if amount < 0:
        raise ValueError("Amount must be positive")
    if amount < threshold:
        return 0
    rate = 0.1 if is_vip else 0.05
    return int(amount * rate)
```

### REFACTOR — 优化

```python
# 重构后
from enum import Enum

class DiscountRate(Enum):
    VIP = 0.10
    REGULAR = 0.05

def calculate_discount(
    amount: float,
    threshold: float,
    is_vip: bool = False,
) -> int:
    _validate_amount(amount)
    if amount < threshold:
        return 0
    rate = DiscountRate.VIP.value if is_vip else DiscountRate.REGULAR.value
    return int(amount * rate)

def _validate_amount(amount: float) -> None:
    if amount < 0:
        raise ValueError("Amount must be positive")
```

---

## Mock 外部依赖

### 使用 unittest.mock

```python
from unittest.mock import patch, AsyncMock, MagicMock
import pytest


# 同步 Mock — 装饰器方式
@patch("module.external_api_call")
def test_with_decorator_mock(mock_api):
    mock_api.return_value = {"status": "ok"}
    result = my_function()
    assert result.is_success


# 同步 Mock — 上下文管理器方式
def test_with_context_mock():
    with patch("module.external_api_call") as mock_api:
        mock_api.return_value = {"status": "ok"}
        result = my_function()
        mock_api.assert_called_once_with(expected_args)


# 异步 Mock
@pytest.mark.asyncio
async def test_async_mock():
    with patch("module.async_fetch", new_callable=AsyncMock) as mock:
        mock.return_value = {"data": []}
        result = await my_async_function()
```

### 使用 pytest-mock（简化写法）

```python
def test_with_mocker(mocker):
    mock_api = mocker.patch("module.external_api_call")
    mock_api.return_value = {"status": "ok"}

    result = my_function()

    mock_api.assert_called_once()
    assert result.is_success
```

### Mock 属性和副作用

```python
def test_side_effect(mocker):
    mock_db = mocker.patch("module.Database")
    # 让第一次调用返回值，第二次抛异常
    mock_db.return_value.query.side_effect = [
        [{"id": 1, "name": "Test"}],
        ConnectionError("DB down"),
    ]
```

---

## pytest fixture 模式

```python
import pytest

# 基础 fixture
@pytest.fixture
def sample_user():
    return {"id": 1, "name": "Test", "role": "user"}

# fixture 依赖其他 fixture
@pytest.fixture
def auth_client(client, sample_user):
    client.force_login(sample_user)
    return client

# 每个测试独立实例（默认就是 function scope）
@pytest.fixture
def fresh_db():
    db = create_test_db()
    yield db          # 测试运行
    db.cleanup()      # 清理（teardown）

# 使用 fixture
def test_user_name(sample_user):
    assert sample_user["name"] == "Test"

def test_authenticated(auth_client):
    response = auth_client.get("/profile")
    assert response.status_code == 200
```

### fixture scope

| scope | 生命周期 | 适用场景 |
|-------|---------|---------|
| `function` | 每个测试（默认） | 大多数场景 |
| `class` | 每个测试类 | 共享 setup |
| `module` | 每个模块 | 数据库连接 |
| `session` | 整个测试会话 | 全局配置 |

---

## 参数化测试

```python
@pytest.mark.parametrize("amount,threshold,expected", [
    (50, 100, 0),       # 低于阈值
    (100, 100, 0),      # 等于阈值
    (200, 100, 10),     # 普通用户
    pytest.param(-1, 100, None, marks=pytest.mark.xfail(raises=ValueError)),
])
def test_calculate_discount_parametrized(amount, threshold, expected):
    assert calculate_discount(amount, threshold) == expected
```

---

## 运行命令

```bash
# 运行所有测试
pytest

# 运行单个文件
pytest tests/test_pricing.py

# 运行匹配的测试
pytest -k "calculate_discount"

# 详细输出
pytest -v

# 显示 print 输出
pytest -s

# 覆盖率报告
pytest --cov=src --cov-report=html

# 只运行失败的测试
pytest --lf

# 并行运行（需安装 pytest-xdist）
pytest -n auto
```
