# TDD 实践模式 — Rust

#编程语言/Rust #软件开发方法论/TDD

---

## 工具链

| 工具 | 用途 | 安装 |
|------|------|------|
| `cargo test` | 内置测试运行器 | Rust 自带 |
| `assert!` / `assert_eq!` | 内置断言宏 | Rust 自带 |
| `mockall` | Mock 框架 | `cargo add mockall --dev` |
| `cargo-tarpaulin` | 覆盖率 | `cargo install cargo-tarpaulin` |
| `nextest` | 增强测试运行器 | `cargo install cargo-nextest` |

### Cargo.toml 配置

```toml
[dev-dependencies]
mockall = "0.12"
assert_matches = "1.5"  # 模式匹配断言（可选）
```

---

## 纯函数 TDD 示例

### RED — 写测试

```rust
// src/pricing.rs

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn returns_zero_below_threshold() {
        let result = calculate_discount(50.0, 100.0, false);
        assert_eq!(result, 0);
    }

    #[test]
    fn applies_vip_rate() {
        let result = calculate_discount(200.0, 100.0, true);
        assert_eq!(result, 20);
    }

    #[test]
    fn applies_regular_rate() {
        let result = calculate_discount(200.0, 100.0, false);
        assert_eq!(result, 10);
    }

    #[test]
    #[should_panic(expected = "Amount must be positive")]
    fn panics_on_negative_amount() {
        calculate_discount(-1.0, 100.0, false);
    }
}
```

### GREEN — 最简实现

```rust
// src/pricing.rs

pub fn calculate_discount(amount: f64, threshold: f64, is_vip: bool) -> i32 {
    if amount < 0.0 {
        panic!("Amount must be positive");
    }
    if amount < threshold {
        return 0;
    }
    let rate = if is_vip { 0.10 } else { 0.05 };
    (amount * rate).floor() as i32
}
```

### REFACTOR — 使用 Result 替代 panic

```rust
// 重构后：用 Result 表达错误，更符合 Rust 惯例
#[derive(Debug, PartialEq)]
pub enum PricingError {
    NegativeAmount,
}

pub fn calculate_discount(
    amount: f64,
    threshold: f64,
    is_vip: bool,
) -> Result<i32, PricingError> {
    if amount < 0.0 {
        return Err(PricingError::NegativeAmount);
    }
    if amount < threshold {
        return Ok(0);
    }
    let rate = if is_vip { Rate::Vip } else { Rate::Regular };
    Ok((amount * rate.value()).floor() as i32)
}

enum Rate {
    Vip,
    Regular,
}

impl Rate {
    fn value(&self) -> f64 {
        match self {
            Rate::Vip => 0.10,
            Rate::Regular => 0.05,
        }
    }
}

// 测试同步更新
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn returns_zero_below_threshold() -> Result<(), PricingError> {
        assert_eq!(calculate_discount(50.0, 100.0, false)?, 0);
        Ok(())
    }

    #[test]
    fn negative_amount_returns_error() {
        assert_eq!(
            calculate_discount(-1.0, 100.0, false),
            Err(PricingError::NegativeAmount)
        );
    }
}
```

---

## Trait Mock 模式（Rust 核心模式）

Rust 通过 **trait** 实现依赖注入和 Mock，是 Rust TDD 的核心技巧。

### 定义 trait

```rust
// src/user/repository.rs
pub trait UserRepository {
    fn find_by_id(&self, id: &str) -> Option<User>;
    fn save(&mut self, user: User) -> Result<(), String>;
}
```

### 使用 mockall 生成 Mock

```rust
// src/user/service.rs
use mockall::automock;

#[cfg_attr(test, automock)]
pub trait UserRepository {
    fn find_by_id(&self, id: &str) -> Option<User>;
    fn save(&mut self, user: User) -> Result<(), String>;
}

pub struct UserService<R: UserRepository> {
    repo: R,
}

impl<R: UserRepository> UserService<R> {
    pub fn new(repo: R) -> Self {
        Self { repo }
    }

    pub fn get_user(&self, id: &str) -> Result<User, String> {
        self.repo.find_by_id(id)
            .ok_or_else(|| format!("User {} not found", id))
    }
}
```

### 测试中使用 Mock

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use mockall::predicate::*;

    #[test]
    fn get_user_found() {
        let mut mock_repo = MockUserRepository::new();
        mock_repo
            .expect_find_by_id()
            .with(eq("123"))
            .returning(|_| Some(User::new("123", "Test")));

        let service = UserService::new(mock_repo);
        let user = service.get_user("123").unwrap();

        assert_eq!(user.name(), "Test");
    }

    #[test]
    fn get_user_not_found() {
        let mut mock_repo = MockUserRepository::new();
        mock_repo
            .expect_find_by_id()
            .with(eq("999"))
            .returning(|_| None);

        let service = UserService::new(mock_repo);
        let result = service.get_user("999");

        assert!(result.is_err());
        assert!(result.unwrap_err().contains("not found"));
    }

    #[test]
    fn save_user_success() {
        let mut mock_repo = MockUserRepository::new();
        mock_repo
            .expect_save()
            .with(predicate::always())
            .returning(|_| Ok(()));

        mock_repo.save(User::new("1", "Test")).unwrap();
    }
}
```

---

## 手写 Mock（简单场景，无需 mockall）

```rust
struct MockUserRepo {
    user: Option<User>,
    save_called: bool,
}

impl MockUserRepo {
    fn new(user: Option<User>) -> Self {
        Self { user, save_called: false }
    }
}

impl UserRepository for MockUserRepo {
    fn find_by_id(&self, _id: &str) -> Option<User> {
        self.user.clone()
    }

    fn save(&mut self, _user: User) -> Result<(), String> {
        self.save_called = true;
        Ok(())
    }
}

#[test]
fn get_user_with_hand_mock() {
    let repo = MockUserRepo::new(Some(User::new("1", "Test")));
    let service = UserService::new(repo);

    assert!(service.get_user("1").is_ok());
}
```

---

## 异步测试

```rust
// Cargo.toml 添加
// [dev-dependencies]
// tokio = { version = "1", features = ["macros", "rt-multi-thread"] }

#[tokio::test]
async fn async_fetch_user() {
    let service = UserService::new(mock_async_repo()).await;
    let user = service.get_user("123").await.unwrap();
    assert_eq!(user.name(), "Test");
}

// 异步 trait Mock（使用 async-trait + mockall）
// #[cfg_attr(test, automock)]
// #[async_trait]
// pub trait AsyncUserRepository {
//     async fn find_by_id(&self, id: &str) -> Option<User>;
// }
```

---

## 测试组织

```rust
// 单元测试：放在同文件 mod tests 中
#[cfg(test)]
mod tests {
    use super::*;
    // 测试私有函数（同一模块内可访问）
}

// 集成测试：放在 tests/ 目录
// tests/integration_test.rs
use my_crate::PricingService;

#[test]
fn integration_calculate_discount() {
    assert_eq!(PricingService::calculate_discount(200.0, 100.0, true), 20);
}

// 文档测试：写在 /// 注释里，既是文档也是测试
/// 计算折扣金额
///
/// # Examples
/// ```
/// use my_crate::calculate_discount;
/// assert_eq!(calculate_discount(200.0, 100.0, false), 10);
/// ```
pub fn calculate_discount(...) -> ... { }
```

---

## 运行命令

```bash
# 运行所有测试
cargo test

# 运行单个模块的测试
cargo test pricing::tests

# 按名称匹配
cargo test calculate_discount

# 显示 println! 输出
cargo test -- --nocapture

# 运行被忽略的测试
cargo test -- --ignored

# 忽略耗时测试（在测试上加 #[ignore]）
#[test]
#[ignore]  // cargo test 默认跳过，cargo test -- --ignored 单独跑
fn slow_integration_test() { ... }

# 覆盖率
cargo tarpaulin
cargo tarpaulin --out Html

# 使用 nextest（更快、更好输出）
cargo nextest run
cargo nextest run -p pricing
```
