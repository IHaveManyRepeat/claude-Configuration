# Rust 安全编码模式

> 本文档是 [[安全概述]] 的语言特定补充，定义 Rust 项目开发中的安全编码规范。

## 内存安全（Rust 天然优势）

### 所有权系统

```rust
// ✅ Good - Rust 所有权系统天然防止缓冲区溢出
fn process_string(s: String) -> usize {
    s.len()  // s 在这里被消耗
}

// ✅ Good - 使用引用避免所有权转移
fn process_str(s: &str) -> usize {
    s.len()  // 只借用，不获取所有权
}

// ❌ Bad - 避免不安全的代码块
unsafe {
    let ptr = malloc(100);
    // 需要手动管理内存，容易出错
}
```

### 避免 unsafe

```rust
// ✅ Good - 优先使用安全抽象
let bytes = &[0u8; 4];
let slice: &[u8] = bytes;

// ❌ Bad - 尽量避免 unsafe
unsafe {
    let ptr: *mut u8 = std::mem::transmute(&mut value);
}
```

## 错误处理

### 使用 Result 而非 panic

```rust
use std::fs::File;
use std::io::{self, Read};

// ✅ Good - 使用 Result 处理错误
fn read_config(path: &str) -> Result<Config, io::Error> {
    let mut file = File::open(path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    let config: Config = toml::from_str(&contents)
        .map_err(|e| io::Error::new(io::ErrorKind::InvalidData, e))?;
    Ok(config)
}

// ✅ Good - 使用 thiserror 定义自定义错误
use thiserror::Error;

#[derive(Error, Debug)]
pub enum AppError {
    #[error("user not found: {0}")]
    NotFound(String),
    
    #[error("validation error: {0}")]
    Validation(String),
    
    #[error("database error: {0}")]
    Database(#[from] sqlx::Error),
}

// ✅ Good - 使用 anyhow 做应用级错误
use anyhow::{Context, Result};

fn process_data(path: &str) -> Result<()> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("Failed to read {}", path))?;
    // ...
    Ok(())
}
```

### 避免 unwrap

```rust
// ❌ Bad - 使用 unwrap 可能 panic
let value = user_input.parse::<i32>().unwrap();

// ✅ Good - 使用 ? 传播错误
let value: i32 = user_input.parse()
    .map_err(|_| MyError::InvalidInput)?;

// ✅ Good - 使用 unwrap_or 提供默认值
let value = user_input.parse::<i32>().unwrap_or(0);

// ✅ Good - 使用 if let 处理 Option
if let Some(user) = find_user(id) {
    println!("Found: {}", user.name);
}
```

## SQL 注入防护

### 使用参数化查询

```rust
// ✅ Good - 使用 sqlx 参数化查询
use sqlx::Row;

async fn get_user(pool: &SqlPool, id: i64) -> Result<Option<User>, sqlx::Error> {
    sqlx::query_as::<_, UserRow>(
        "SELECT id, name, email FROM users WHERE id = ?"
    )
    .bind(id)  // 参数绑定
    .fetch_optional(pool)
    .await
}

// ✅ Good - 使用 sea-orm
use sea_orm::EntityTrait;

let user = User::find_by_id(42).one(&db).await?;
```

## 密码存储

```rust
use bcrypt::{hash, verify, DEFAULT_COST};

// ✅ Good - 使用 bcrypt 哈希密码
fn hash_password(password: &str) -> String {
    hash(password, DEFAULT_COST).unwrap()
}

fn verify_password(password: &str, hash: &str) -> bool {
    verify(password, hash).unwrap()
}
```

## 密钥管理

```rust
use std::env;

// ✅ Good - 从环境变量读取
fn get_api_key() -> Result<String, env::VarError> {
    env::var("API_KEY")
}

// ✅ Good - 使用 dotenvy 加载 .env
use dotenvy::dotenv;
dotenv().ok();

fn get_database_url() -> String {
    env::var("DATABASE_URL").expect("DATABASE_URL must be set")
}

// ❌ Bad - 硬编码密钥
const API_KEY: &str = "sk-1234567890";  // 危险！
```

## 并发安全

```rust
use std::sync::{Arc, Mutex};
use std::thread;

// ✅ Good - 使用 Arc<Mutex<T>> 保护共享状态
let counter = Arc::new(Mutex::new(0));
let mut handles = vec![];

for _ in 0..10 {
    let counter = Arc::clone(&counter);
    let handle = thread::spawn(move || {
        let mut num = counter.lock().unwrap();
        *num += 1;
    });
    handles.push(handle);
}

// ✅ Good - 使用 tokio 异步运行时
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let (a, b) = tokio::join!(async_a(), async_b());
    Ok(())
}
```

## 依赖安全

```bash
# 定期运行安全审计
cargo audit

# 使用 cargo deny 检查依赖
cargo install cargo-deny
cargo deny check

# 检查漏洞
cargo install cargo-geiger
cargo geiger
```

## 安全检查清单

- [ ] 避免使用 unsafe 代码块
- [ ] 使用 Result 处理错误而非 panic
- [ ] SQL 使用参数化查询
- [ ] 密码使用 bcrypt 存储
- [ ] 密钥从环境变量读取，无硬编码
- [ ] 并发使用 Arc<Mutex<T>> 或 channel
- [ ] 定期运行 cargo audit、cargo deny

## 参考资源

- [Rust 安全指南](https://rust-lang.github.io/api-guidelines/security.html)
- [OWASP Rust 速查表](https://cheatsheetseries.owasp.org/cheatsheets/Rust_Security_Cheat_Sheet.html)
- [Rust  cookbook 安全示例](https://rust-lang-nursery.github.io/rust-cookbook/security)