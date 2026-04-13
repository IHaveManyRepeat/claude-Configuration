# JavaScript 最佳实践

## 变量与常量

### 优先使用 `const`，需要重赋值时用 `let`，永不使用 `var`

```javascript
// ❌ Bad
var count = 0;
var name = "Alice";

// ✅ Good
const MAX_RETRY = 3;
let currentCount = 0;
```

### 解构赋值

```javascript
// ✅ 对象解构 + 重命名 + 默认值
const { name: userName, age = 18, ...rest } = user;

// ✅ 数组解构 + 跳过
const [, second, third] = results;

// ✅ 函数参数解构
function createUser({ name, email, role = "user" }) {
  return { name, email, role };
}
```

## 函数

### 箭头函数 vs 普通函数

```javascript
// ✅ 箭头函数：纯函数、回调、不依赖 this
const add = (a, b) => a + b;
const names = users.map(u => u.name);

// ✅ 普通函数：需要 this、arguments、作为构造函数
class UserService {
  constructor() {
    this.cache = new Map();
  }

  // 方法中使用普通函数或类方法
  getUser(id) {
    return this.cache.get(id);
  }
}
```

### 默认参数

```javascript
// ❌ Bad
function config(options) {
  const timeout = options.timeout || 5000;
  const retries = options.retries || 3;
}

// ✅ Good
function config({ timeout = 5000, retries = 3 } = {}) {
  return { timeout, retries };
}
```

### Rest/Spread 操作

```javascript
// ✅ 不可变更新
const updatedUser = { ...user, name: "New Name" };
const addedItems = [...items, newItem];
const removedItems = items.filter((_, i) => i !== index);

// ✅ 合并配置
const mergedConfig = { ...defaultConfig, ...userConfig };
```

## 异步编程

### async/await 是首选

```javascript
// ❌ Bad - 回调地狱
function fetchUser(id, callback) {
  db.find(id, (err, user) => {
    if (err) return callback(err);
    cache.set(user, (err) => {
      if (err) return callback(err);
      callback(null, user);
    });
  });
}

// ✅ Good - async/await
async function fetchUser(id) {
  const user = await db.find(id);
  await cache.set(user);
  return user;
}
```

### 并行执行独立异步操作

```javascript
// ❌ Bad - 串行等待
const users = await fetchUsers();
const posts = await fetchPosts();

// ✅ Good - 并行
const [users, posts] = await Promise.all([
  fetchUsers(),
  fetchPosts(),
]);

// ✅ 全部完成（不因单个失败而中断）
const results = await Promise.allSettled([
  fetchFromAPI("a"),
  fetchFromAPI("b"),
  fetchFromAPI("c"),
]);
```

### 错误处理

```javascript
// ✅ try-catch 包裹可能失败的异步操作
async function safeFetch(url) {
  try {
    const response = await fetch(url);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return await response.json();
  } catch (error) {
    console.error(`Failed to fetch ${url}:`, error);
    return null;
  }
}
```

## 数组操作

### 使用函数式方法替代命令式循环

```javascript
// ❌ Bad
const names = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].active) {
    names.push(users[i].name);
  }
}

// ✅ Good
const names = users
  .filter(user => user.active)
  .map(user => user.name);

// ✅ 常用数组方法
const found = users.find(u => u.id === targetId);     // 查找单个
const hasAdmin = users.some(u => u.role === "admin");  // 存在判断
const allActive = users.every(u => u.active);          // 全部满足
const grouped = users.reduce((acc, u) => {             // 分组
  (acc[u.role] ??= []).push(u);
  return acc;
}, {});
```

### 数组不可变操作

```javascript
// ✅ 添加
const added = [...arr, newItem];
// ✅ 插入
const inserted = [...arr.slice(0, i), newItem, ...arr.slice(i)];
// ✅ 删除
const removed = [...arr.slice(0, i), ...arr.slice(i + 1)];
// ✅ 替换
const replaced = arr.map((item, idx) => idx === i ? newItem : item);
// ✅ 排序（不修改原数组）
const sorted = [...arr].sort((a, b) => a - b);
```

## 对象操作

```javascript
// ✅ 动态属性名
const key = "name";
const obj = { [key]: "Alice" };

// ✅ 可选链
const city = user?.address?.city ?? "Unknown";

// ✅ Object 静态方法
Object.keys(obj);      // 键数组
Object.values(obj);    // 值数组
Object.entries(obj);   // [key, value] 数组
Object.fromEntries();  // 从 [key, value] 数组创建对象

// ✅ 深拷贝（现代浏览器）
const cloned = structuredClone(original);
```

## 模块化

```javascript
// ✅ 命名导出为主
export function formatDate(date) { /* ... */ }
export const DEFAULT_FORMAT = "YYYY-MM-DD";

// ✅ 默认导出仅用于模块主功能
export default class UserService { /* ... */ }

// ✅ 按需导入
import { formatDate, DEFAULT_FORMAT } from "./utils.js";
```

## 常见陷阱

### 避免隐式类型转换

```javascript
// ❌ Bad
if (value) { }          // 隐式布尔转换
if (a == b) { }         // 宽松相等

// ✅ Good
if (value !== null && value !== undefined) { }
if (value != null) { }  // 仅 null/undefined 检查的简写
if (a === b) { }        // 严格相等
```

### 正确判断空值

```javascript
// ✅ null/undefined 判断
if (value == null) { }      // 覆盖 null 和 undefined
if (value === undefined) { } // 仅 undefined

// ✅ 空数组判断
if (arr.length === 0) { }

// ✅ 空对象判断
if (Object.keys(obj).length === 0) { }
```

## 性能建议

- 使用 `Map`/`Set` 替代普通对象做频繁增删的键值存储
- 大量字符串拼接用模板字面量 `` `${a}${b}` `` 而非 `+`
- 避免在循环中创建函数
- 使用 `requestAnimationFrame` 做动画，`setTimeout`/`setInterval` 做定时
- 利用 Web Workers 处理 CPU 密集型任务

## 参考资源

- [MDN JavaScript 指南](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide)
- [JavaScript.info](https://zh.javascript.info/)
- [ES6 标准入门](https://es6.ruanyifeng.com/)
- [Airbnb JavaScript 风格指南](https://github.com/airbnb/javascript)
