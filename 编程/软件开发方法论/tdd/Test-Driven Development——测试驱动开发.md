# Test-Driven Development (TDD) — 测试驱动开发

> **定位**：Claude Code 开发时的 TDD 速查手册。涵盖核心原则、工作流、分语言实践模式和常见反模式。

#软件开发方法论/TDD #核心方法论

---

## 一、TDD 核心循环：Red → Green → Refactor

```
┌─────────────────────────────────────────────┐
│  1. RED    — 写一个会失败的测试              │
│  2. GREEN  — 写最少的代码让测试通过          │
│  3. REFACTOR — 在测试保护下重构              │
│         ↑ 重复循环 ↑                         │
└─────────────────────────────────────────────┘
```

### 每一步的要点

| 阶段 | 目标 | 关键约束 |
|------|------|----------|
| **Red** | 明确需求，写一个描述行为的测试 | 测试必须失败，且失败信息要清晰表达期望 |
| **Green** | 让测试通过 | 只写刚好够用的代码，不过度设计 |
| **Refactor** | 消除重复，改善结构 | 测试必须持续通过，行为不变 |

### 步骤粒度：小步前进

- 每次只测一个行为（一个 `it` / 一个 `test`）
- 每次循环 1-5 分钟，不超过 10 分钟
- 如果 Green 阶段超过 5 分钟，说明 Red 的步子太大了，回退拆分

---

## 二、测试金字塔与分层策略

```
        ╱  E2E  ╲           少量 · 端到端 · 慢 · 脆弱
       ╱─────────╲
      ╱ Integration ╲       适量 · 模块间协作 · 中速
     ╱───────────────╲
    ╱    Unit Tests    ╲     大量 · 单个函数/类 · 快 · 稳定
   ╱───────────────────╲
```

| 层级 | 覆盖目标 | 占比建议 | 速度 |
|------|----------|----------|------|
| **Unit** | 单个函数、类、纯逻辑 | 70-80% | < 10ms |
| **Integration** | 模块交互、数据库、API | 15-20% | < 500ms |
| **E2E** | 用户关键路径 | 5-10% | 秒级 |

### 决策规则

- **业务逻辑** → Unit Test（必须 TDD）
- **API 端点** → Integration Test（建议 TDD）
- **数据库查询** → Integration Test（可后补）
- **用户操作流** → E2E Test（关键路径覆盖）

---

## 三、AAA 模式（Arrange-Act-Assert）

所有测试统一使用三段式结构：

```
test('描述被测行为', () => {
  // Arrange — 准备输入和前置条件
  const input = ...;
  const expected = ...;

  // Act — 执行被测操作（一行，明确）
  const result = doSomething(input);

  // Assert — 验证输出/状态变更
  expect(result).toEqual(expected);
});
```

### 命名规范

| 风格 | 格式 | 示例 |
|------|------|------|
| **行为描述** | `should + 期望行为 + when + 条件` | `should return 404 when user not found` |
| **Given-When-Then** | `given X, when Y, then Z` | `given empty cart, when add item, then cart has 1 item` |
| **中文项目** | `当 X 时，应该 Y` | `当用户未登录时，应该返回 401` |

---

## 四、测试替身（Test Doubles）

### 选择决策树

```
需要隔离外部依赖？
├── 只需要控制返回值 → Stub（桩）
├── 需要验证是否被调用 → Mock（模拟）
├── 需要录制调用信息 → Spy（间谍）
├── 需要简化复杂对象 → Fake（假实现，如内存数据库）
└── 不需要隔离 → 用真实对象
```

### 使用原则

1. **不 Mock 被测系统本身** — 只 Mock 外部依赖
2. **优先 Stub 而非 Mock** — 验证输出优于验证调用
3. **Mock 最小化** — 每个 Mock 只服务于当前测试
4. **Mock 放在 Arrange 阶段** — 不要在 Assert 中设置 Mock

---

## 五、分语言实践模式

各语言详细 TDD 模式和代码示例，按语言独立文件：

- [[分语言实践模式/TypeScript]] — Vitest / Jest，纯函数、Mock、React 组件测试
- [[分语言实践模式/Python]] — pytest，fixture、参数化、异步 Mock
- [[分语言实践模式/Go]] — testing + testify，接口 Mock、表驱动测试、HTTP Handler
- [[分语言实践模式/JavaScript]] — Vitest / Jest，纯函数、Mock、Node.js HTTP 接口测试
- [[分语言实践模式/Java]] — JUnit 5 / AssertJ / Mockito，依赖注入、参数化测试、Spring Boot 集成测试
- [[分语言实践模式/Rust]] — 内置 #[test]，trait Mock、async 测试、Result 错误处理模式

---

## 六、设计原则与 TDD 的关系

### 可测试性 = 好设计

| 可测试的代码特征 | 对应设计原则 |
|------------------|-------------|
| 依赖可注入 | 依赖倒置（DIP） |
| 函数纯（无副作用） | 单一职责（SRP） |
| 不依赖全局状态 | 无副作用设计 |
| 接口小而专 | 接口隔离（ISP） |

### 如果测试难写，说明设计有问题

| 测试困难 | 根因 | 解决方案 |
|----------|------|----------|
| 需要大量 setup | 职责过多 | 拆分为更小的函数/类 |
| Mock 链太长 | 耦合过紧 | 引入接口解耦 |
| 测试随机失败 | 隐式依赖（时间、随机数、全局状态） | 将依赖作为参数注入 |
| 需要测私有方法 | 方法不应私有 | 提取为新类 |

---

## 七、常见反模式

### ❌ 避免

| 反模式 | 问题 | 正确做法 |
|--------|------|----------|
| 先写实现再补测试 | 不是 TDD，测试只验证已写代码 | 严格 Red → Green → Refactor |
| 一个测试测多个行为 | 失败时无法定位问题 | 一个 `it` / `test` 只测一件事 |
| Mock 被测系统自身 | 测的是 Mock 不是逻辑 | 只 Mock 外部依赖 |
| 测试间共享可变状态 | 顺序依赖，难以并行 | 每个测试独立 setup/teardown |
| 测试实现细节 | 重构会破坏测试 | 测试公开行为，不测私有方法 |
| `expect.any()` 泛匹配 | 测了等于没测 | 精确断言期望值 |
| 超过 3 层嵌套 describe | 结构混乱 | 扁平化或拆文件 |

### ✅ 检查清单

- [ ] 测试名称描述行为而非实现（"should return 404" 而非 "should call db"）
- [ ] 每个测试可独立运行（无执行顺序依赖）
- [ ] Arrange 阶段清晰，不隐藏关键输入
- [ ] 只有一个 Assert 主题（可以多个断言，但验证同一件事）
- [ ] 无条件跳过的测试（`skip`）不超过 5%，且附带 TODO
- [ ] 测试运行时间：Unit < 10ms，Integration < 500ms

---

## 八、与 Claude Code 协作 TDD 的工作流

### 推荐流程

```
1. 需求 → 描述为测试用例列表（Claude 辅助拆分）
2. 逐个执行 Red → Green → Refactor 循环
3. 每完成一个循环，Claude 可执行 code-review
4. 全部通过后，检查覆盖率（目标 ≥ 80%）
```

### 给 Claude 的典型指令

```
# 启动 TDD
用 TDD 方式实现 [功能描述]，先写测试再写实现

# 继续 TDD 循环
写下一个测试用例，然后实现让它通过

# 代码审查
review 刚才写的代码和测试，关注可测试性和边界情况

# 覆盖率检查
运行测试覆盖率报告，找出未覆盖的分支
```

---

## 九、参考工具速查

| 语言 | 测试框架 | 断言库 | Mock 工具 | 覆盖率 | 详细模式 |
|------|---------|--------|----------|--------|---------|
| TypeScript | Vitest / Jest | 内置 | vi.mock() / jest.fn() | c8 / v8 | [[分语言实践模式/TypeScript]] |
| Python | pytest | 内置 assert | unittest.mock / pytest-mock | pytest-cov | [[分语言实践模式/Python]] |
| Go | testing + testify | testify/assert | testify/mock / 手写 | go test -cover | [[分语言实践模式/Go]] |
| JavaScript | Vitest / Jest | 内置 | vi.mock() / jest.fn() | c8 / v8 | [[分语言实践模式/JavaScript]] |
| Java | JUnit 5 | AssertJ | Mockito | JaCoCo | [[分语言实践模式/Java]] |
| Rust | 内置 #[test] | assert! / assert_eq! | mockall | cargo tarpaulin | [[分语言实践模式/Rust]] |
| C# | xUnit / NUnit | FluentAssertions | Moq | coverlet | |
