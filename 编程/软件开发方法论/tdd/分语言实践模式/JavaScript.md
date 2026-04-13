# TDD 实践模式 — JavaScript

#编程语言/JavaScript #软件开发方法论/TDD

---

## 工具链

| 工具 | 用途 | 安装 |
|------|------|------|
| Vitest（推荐） | 测试框架 | `npm i -D vitest` |
| Jest | 测试框架（备选） | `npm i -D jest` |
| happy-dom | DOM 模拟 | `npm i -D happy-dom` |
| c8 / v8 | 覆盖率 | `npm i -D @vitest/coverage-v8` |
| supertest | HTTP 测试 | `npm i -D supertest` |

> **注意**：JavaScript 与 TypeScript 共享 Vitest/Jest 工具链，本文件侧重纯 JS 项目的写法差异。TS 项目详见 [[分语言实践模式/TypeScript]]。

### vitest.config.js（CommonJS）

```javascript
const { defineConfig } = require('vitest/config');

module.exports = defineConfig({
  test: {
    globals: true,
    environment: 'happy-dom',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: ['node_modules/', 'tests/', '**/*.config.*'],
    },
  },
});
```

### vitest.config.js（ESM）

```javascript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
  },
});
```

---

## 纯函数 TDD 示例

### RED — 写测试

```javascript
// tests/pricing.test.js
import { describe, it, expect } from 'vitest';
import { calculateDiscount } from '../src/pricing.js';

describe('calculateDiscount', () => {
  it('should return 0 when amount is below threshold', () => {
    expect(calculateDiscount(50, 100)).toBe(0);
  });

  it('should apply 10% discount for VIP users', () => {
    expect(calculateDiscount(200, 100, { isVIP: true })).toBe(20);
  });

  it('should apply 5% discount for regular users above threshold', () => {
    expect(calculateDiscount(200, 100)).toBe(10);
  });

  it('should throw when amount is negative', () => {
    expect(() => calculateDiscount(-1, 100)).toThrow('Amount must be positive');
  });

  it('should return 0 when amount equals threshold', () => {
    expect(calculateDiscount(100, 100)).toBe(0);
  });
});
```

### GREEN — 最简实现

```javascript
// src/pricing.js
export function calculateDiscount(amount, threshold, options = {}) {
  if (amount < 0) throw new Error('Amount must be positive');
  if (amount < threshold) return 0;
  const rate = options.isVIP ? 0.1 : 0.05;
  return Math.floor(amount * rate);
}
```

### REFACTOR — 优化

```javascript
// src/pricing.js
const DISCOUNT_RATES = { vip: 0.10, regular: 0.05 };

export function calculateDiscount(amount, threshold, options = {}) {
  validateAmount(amount);
  if (amount < threshold) return 0;
  return Math.floor(amount * getRate(options.isVIP));
}

function validateAmount(amount) {
  if (amount < 0) throw new Error('Amount must be positive');
}

function getRate(isVIP) {
  return isVIP ? DISCOUNT_RATES.vip : DISCOUNT_RATES.regular;
}
```

---

## Mock 模式

### Mock 模块

```javascript
import { vi } from 'vitest';

// Mock 整个模块
vi.mock('../src/api.js', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: 1, name: 'Test' }),
}));

// 在测试中动态切换返回值
it('should handle different responses', async () => {
  const { fetchUser } = await import('../src/api.js');

  fetchUser.mockResolvedValueOnce({ id: 1, name: 'Alice' });
  fetchUser.mockRejectedValueOnce(new Error('Network error'));

  const r1 = await fetchUser('1');
  expect(r1.name).toBe('Alice');

  await expect(fetchUser('2')).rejects.toThrow('Network error');
});
```

### Mock 定时器

```javascript
import { vi } from 'vitest';

describe('debounce', () => {
  beforeEach(() => { vi.useFakeTimers(); });
  afterEach(() => { vi.restoreAllMocks(); vi.useRealTimers(); });

  it('should delay execution', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 100);

    debounced();
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(50);
    debounced();
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(100);
    expect(fn).toHaveBeenCalledOnce();
  });
});
```

### Spy — 监听函数调用

```javascript
it('should call callback with result', () => {
  const callback = vi.fn();

  processData({ value: 42 }, callback);

  expect(callback).toHaveBeenCalledWith({ value: 42, processed: true });
  expect(callback).toHaveBeenCalledOnce();
});
```

---

## Node.js HTTP 接口测试

```javascript
// 使用 supertest 测试 Express 应用
import { describe, it, expect, beforeEach } from 'vitest';
import request from 'supertest';
import { createApp } from '../src/app.js';

describe('GET /api/users/:id', () => {
  let app;

  beforeEach(() => {
    app = createApp({ db: createTestDb() });
  });

  it('should return user when found', async () => {
    const res = await request(app).get('/api/users/123');

    expect(res.status).toBe(200);
    expect(res.body).toEqual({ id: '123', name: 'Test' });
  });

  it('should return 404 when not found', async () => {
    const res = await request(app).get('/api/users/999');

    expect(res.status).toBe(404);
    expect(res.body.error).toBe('User not found');
  });
});
```

---

## 测试异步代码

```javascript
// async/await — Promise 成功
it('should fetch data', async () => {
  const data = await fetchData('123');
  expect(data.id).toBe('123');
});

// async/await — Promise 拒绝
it('should handle rejection', async () => {
  await expect(fetchData('999')).rejects.toThrow('Not found');
});

// callback — 用 vi.fn() 代替 done()
it('should invoke callback', () => {
  const cb = vi.fn();
  processData(input, cb);
  expect(cb).toHaveBeenCalledWith(expectedResult);
});

// 事件监听
it('should emit complete event', () => {
  const emitter = new EventEmitter();
  const handler = vi.fn();
  emitter.on('complete', handler);

  emitter.emit('complete', { status: 'done' });

  expect(handler).toHaveBeenCalledWith({ status: 'done' });
});
```

---

## DOM 测试（浏览器环境代码）

```javascript
// vitest.config.js 设置 environment: 'happy-dom'

it('should update counter display on increment', () => {
  const container = document.createElement('div');
  container.setAttribute('id', 'app');
  document.body.appendChild(container);

  const counter = createCounter(container);
  counter.increment();

  const countEl = container.querySelector('#count');
  expect(countEl.textContent).toBe('1');
});
```

---

## 运行命令

```bash
# 运行所有测试
npx vitest run

# 监听模式
npx vitest

# 运行单个文件
npx vitest run tests/pricing.test.js

# 按名称匹配
npx vitest run -t "calculateDiscount"

# 覆盖率报告
npx vitest run --coverage

# 只运行变更的文件相关测试
npx vitest run --changed

# 并行/串行
npx vitest run --pool=forks        # fork 进程池（默认）
npx vitest run --no-threads        # 单线程（调试用）
```
