# Python 安全编码模式

> 本文档是 [[安全概述]] 的语言特定补充，定义 Python 项目开发中的安全编码规范。

## 输入验证

### 使用 Pydantic 模型验证

```python
from pydantic import BaseModel, EmailStr, validator, Field
from typing import Optional

class UserCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr  # 自动验证邮箱格式
    age: int = Field(..., ge=0, le=150)
    
    @validator('name')
    def validate_name(cls, v: str) -> str:
        # 移除潜在的危险字符
        return v.strip()

# ✅ 自动验证输入
user = UserCreate(name="  Alice  ", email="alice@example.com", age=25)
# name 会被 strip，email 会验证格式，age 会在范围内
```

### 防止 SQL 注入

```python
# ✅ Good - 使用参数化查询
import sqlite3

def get_user_by_id(user_id: int) -> Optional[dict]:
    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    
    # 参数化查询
    cursor.execute(
        "SELECT id, name, email FROM users WHERE id = ?",
        (user_id,)
    )
    row = cursor.fetchone()
    conn.close()
    
    if row:
        return {"id": row[0], "name": row[1], "email": row[2]}
    return None

# ❌ Bad - 字符串拼接（SQL 注入风险）
def get_user_by_id(user_id: int) -> Optional[dict]:
    conn = sqlite3.connect('app.db')
    cursor = conn.cursor()
    
    # 危险！用户输入直接拼接到 SQL
    cursor.execute(f"SELECT id, name, email FROM users WHERE id = {user_id}")
```

### 使用 ORM（推荐）

```python
# ✅ Good - 使用 SQLAlchemy ORM
from sqlalchemy import select
from sqlalchemy.orm import Session

def get_user_by_id(session: Session, user_id: int) -> Optional[User]:
    return session.execute(
        select(User).where(User.id == user_id)
    ).scalar_one_or_none()
```

## 密码存储

```python
import bcrypt

# ✅ Good - 使用 bcrypt 哈希密码
def hash_password(password: str) -> str:
    salt = bcrypt.gensalt()
    return bcrypt.hashpw(password.encode('utf-8'), salt).decode('utf-8')

def verify_password(password: str, hashed: str) -> bool:
    return bcrypt.checkpw(
        password.encode('utf-8'),
        hashed.encode('utf-8')
    )
```

## 密钥管理

```python
import os
from functools import lru_cache

# ✅ Good - 从环境变量读取
@lru_cache()
def get_api_key() -> str:
    api_key = os.environ.get("API_KEY")
    if not api_key:
        raise ValueError("API_KEY environment variable is required")
    return api_key

# ✅ Good - 使用 pydantic-settings
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str
    secret_key: str
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"

settings = Settings()

# ❌ Bad - 硬编码密钥
API_KEY = "sk-1234567890abcdef"  # 危险！
```

## 序列化安全

```python
import pickle
import json

# ❌ Bad - pickle 反序列化不可信数据
def load_data(data: bytes):
    # 危险！可能执行任意代码
    return pickle.loads(data)

# ✅ Good - 使用 JSON
def load_data(data: str):
    return json.loads(data)

# ✅ Good - 使用 pydantic
def parse_user(data: dict):
    return User.model_validate(data)
```

## Web 安全

```python
from markupsafe import escape, Markup

# ✅ Good - 输出转义
def render_user_input(user_input: str) -> str:
    # 自动转义 HTML 特殊字符
    return escape(user_input)

# ✅ Good - 选择性信任 HTML
def render_html(html_content: str) -> Markup:
    # 仅对完全可信的 HTML 使用 markupsafe
    return Markup(html_content)
```

## 依赖安全

```bash
# 定期审计依赖漏洞
pip install safety
safety check

# 使用 pip-audit
pip install pip-audit
pip-audit
```

## 安全检查清单

- [ ] 使用 Pydantic 验证所有外部输入
- [ ] SQL 使用 ORM 或参数化查询
- [ ] 密码使用 bcrypt 存储
- [ ] 密钥从环境变量读取，无硬编码
- [ ] 不使用 pickle 反序列化不可信数据
- [ ] 输出使用 markupsafe 转义
- [ ] 定期运行 safety、pip-audit

## 参考资源

- [OWASP Python 速查表](https://cheatsheetseries.owasp.org/cheatsheets/Python_Security_Cheat_Sheet.html)
- [Pydantic 安全验证](https://docs.pydantic.dev/)