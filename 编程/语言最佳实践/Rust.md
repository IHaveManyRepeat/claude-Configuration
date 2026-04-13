# Rust 最佳实践

## 所有权与借用

### 理解所有权规则

```rust
// 所有权三规则：
// 1. 每个值有且只有一个所有者
// 2. 同一时刻只能有一个可变引用或多个不可变引用
// 3. 值离开作用域时自动释放

// ✅ 使用引用避免转移所有权
fn calculate_length(s: &str) -> usize {
    s.len()
} // s 是引用，不拥有值，离开作用域无需释放

fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1); // 借用，s1 仍然有效
    println!("'{}' 的长度是 {}", s1, len);
}
```

### 借用规则

```rust
// ✅ 多个不可变引用 —— OK
let s = String::from("hello");
let r1 = &s;
let r2 = &s;
println!("{} {}", r1, r2);

// ✅ 可变引用（同时只有一个）
let mut s = String::from("hello");
let r = &mut s;
r.push_str(", world");

// ✅ 不可变引用的生命周期不能与可变引用重叠
let mut s = String::from("hello");
let r1 = &s;       // 不可变引用
let r2 = &s;       // 不可变引用
println!("{} {}", r1, r2); // r1, r2 最后一次使用
let r3 = &mut s;   // 现在可以可变借用
r3.push_str("!");
```

## 错误处理

### 使用 `Result` 而非 panic

```rust
use std::fs;

// ❌ Bad - panic on error
let content = fs::read_to_string("config.toml").unwrap();

// ✅ Good - 使用 ? 操作符传播错误
fn read_config() -> Result<Config, Box<dyn std::error::Error>> {
    let content = fs::read_to_string("config.toml")?;
    let config: Config = toml::from_str(&content)?;
    Ok(config)
}

// ✅ 使用 thiserror 定义自定义错误
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("user {id} not found")]
    NotFound { id: String },

    #[error("validation failed: {message}")]
    Validation { message: String },

    #[error("database error")]
    Database(#[from] sqlx::Error),
}

// ✅ 使用 anyhow 做应用级错误传播
use anyhow::{Context, Result};

fn process_file(path: &str) -> Result<()> {
    let content = fs::read_to_string(path)
        .with_context(|| format!("Failed to read {}", path))?;
    // ...
    Ok(())
}
```

## 类型设计

### 使用 newtype 模式

```rust
// ✅ newtype 提供类型安全
struct UserId(String);
struct OrderId(String);

fn get_user(id: UserId) -> User { ... }
fn get_order(id: OrderId) -> Order { ... }

// 编译器会阻止混用
// get_user(OrderId("123".into())); // 编译错误！
```

### 使用枚举建模状态

```rust
// ✅ 枚举 + 模式匹配是 Rust 的核心表达力
enum GameState {
    NotStarted,
    InProgress { turn: u32, current_player: Player },
    Finished { winner: Player },
}

impl GameState {
    fn describe(&self) -> &str {
        match self {
            GameState::NotStarted => "Waiting to start",
            GameState::InProgress { turn, .. } => "Game ongoing",
            GameState::Finished { winner } => "Game over",
        }
    }
}

// ✅ Option 表达可能缺失的值
fn find_user(id: u32) -> Option<User> {
    database::get(id)  // 返回 None 或 Some(User)
}

match find_user(id) {
    Some(user) => println!("Found: {}", user.name),
    None => println!("User not found"),
}

// ✅ 使用 if let 简化单个分支
if let Some(user) = find_user(id) {
    println!("Found: {}", user.name);
}
```

### Builder 模式

```rust
// ✅ Builder 模式处理复杂构造
#[derive(Debug)]
struct Server {
    host: String,
    port: u16,
    timeout: std::time::Duration,
    max_connections: usize,
}

struct ServerBuilder {
    host: Option<String>,
    port: Option<u16>,
    timeout: Option<std::time::Duration>,
    max_connections: Option<usize>,
}

impl ServerBuilder {
    fn new() -> Self {
        Self {
            host: None,
            port: None,
            timeout: None,
            max_connections: None,
        }
    }

    fn host(mut self, host: impl Into<String>) -> Self {
        self.host = Some(host.into());
        self
    }

    fn port(mut self, port: u16) -> Self {
        self.port = Some(port);
        self
    }

    fn build(self) -> Result<Server, &'static str> {
        Ok(Server {
            host: self.host.unwrap_or_else(|| "127.0.0.1".to_string()),
            port: self.port.unwrap_or(8080),
            timeout: self.timeout.unwrap_or_else(|| Duration::from_secs(30)),
            max_connections: self.max_connections.unwrap_or(100),
        })
    }
}

// 使用
let server = ServerBuilder::new()
    .host("0.0.0.0")
    .port(3000)
    .build()?;
```

## 并发

### 使用 Channel 消息传递

```rust
use std::sync::mpsc;
use std::thread;

fn main() {
    let (tx, rx) = mpsc::channel();

    thread::spawn(move || {
        let data = fetch_data();
        tx.send(data).unwrap();
    });

    let received = rx.recv().unwrap();
}
```

### 使用 tokio 异步运行时

```rust
use tokio;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 并发执行
    let (users, posts) = tokio::join!(
        fetch_users(),
        fetch_posts(),
    );

    // 有界通道控制背压
    let (tx, mut rx) = tokio::sync::mpsc::channel::<Job>(100);

    // spawn 任务
    for i in 0..10 {
        let tx = tx.clone();
        tokio::spawn(async move {
            let result = process(i).await;
            tx.send(result).await.unwrap();
        });
    }

    drop(tx); // 关闭发送端
    while let Some(result) = rx.recv().await {
        println!("Got: {:?}", result);
    }

    Ok(())
}
```

### Mutex 和 RwLock

```rust
use std::sync::{Arc, Mutex};

// ✅ Arc<Mutex<T>> 共享可变状态
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
```

## Trait 设计

### 默认实现

```rust
// ✅ 提供默认实现
trait Summary {
    fn summarize_author(&self) -> &str;

    // 默认实现
    fn summarize(&self) -> String {
        format!("Read more from {}...", self.summarize_author())
    }
}
```

### 关联类型 vs 泛型

```rust
// ✅ 关联类型：每个实现只有一个具体类型
trait Graph {
    type Node;
    type Edge;
    fn edges(&self, node: &Self::Node) -> Vec<Self::Edge>;
}

// ✅ 泛型 trait：每个类型可以多次实现
trait From<T> {
    fn from(value: T) -> Self;
}
```

## 性能建议

- 使用 `&str` 替代 `String` 作为函数参数（避免不必要的分配）
- 使用 `Cow<str>` 处理"有时需要拥有、有时只需借用"的字符串
- 用 `Vec::with_capacity` 预分配
- 热点路径避免频繁 `clone()`，检查是否可以用引用
- 使用 `cargo bench` 做基准测试
- 使用 `cargo profile` 分析性能瓶颈

## 工具链

```bash
cargo fmt              # 格式化代码
cargo clippy           # lint 检查
cargo test             # 运行测试
cargo bench            # 基准测试
cargo doc --open       # 生成文档
cargo audit            # 安全审计
cargo tree             # 依赖树
```

## 参考资源

- [The Rust Book](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)
- [Rust Design Patterns](https://rust-unofficial.github.io/patterns/)
- [Comprehensive Rust (Google)](https://google.github.io/comprehensive-rust/)
