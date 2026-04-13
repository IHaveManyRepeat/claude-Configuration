# TDD 实践模式 — TypeScript / JavaScript

#编程语言/TypeScript #软件开发方法论/TDD

---

## 工具链

| 工具 | 用途 | 安装 |
|------|------|------|
| Vitest（推荐） | 测试框架 | `npm i -D vitest` |
| Jest | 测试框架（备选） | `npm i -D jest` |
| c8 / v8 | 覆盖率 | `npm i -D @vitest/coverage-v8` |
| happy-dom | DOM 模拟 | `npm i -D happy-dom` |

### vitest.config.ts

```typescript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'happy-dom',  // 需要测 DOM 时使用
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      exclude: ['node_modules/', 'tests/', '**/*.d.ts'],
    },
  },
});
```

---

## 纯函数 TDD 示例

### RED — 写测试

```typescript
// tests/pricing.test.ts
import { describe, it, expect } from 'vitest';
import { calculateDiscount } from '../src/pricing';

describe('calculateDiscount', () => {
  it('should return 0 when amount is below threshold', () => {
    expect(calculateDiscount(50, 100)).toBe(0);
  });

  it('should apply 10% discount for VIP users', () => {
    expect(calculateDiscount(200, 100, { isVIP: true })).toBe(20);
  });

  it('should throw when amount is negative', () => {
    expect(() => calculateDiscount(-1, 100)).toThrow('Amount must be positive');
  });
});
```

### GREEN — 最简实现

```typescript
// src/pricing.ts
export function calculateDiscount(
  amount: number,
  threshold: number,
  options?: { isVIP?: boolean }
) {
  if (amount < 0) throw new Error('Amount must be positive');
  if (amount < threshold) return 0;
  const rate = options?.isVIP ? 0.1 : 0.05;
  return Math.floor(amount * rate);
}
```

### REFACTOR — 优化

```typescript
// 重构后：提取常量，改善可读性
const DISCOUNT_RATES = {
  vip: 0.10,
  regular: 0.05,
} as const;

export function calculateDiscount(
  amount: number,
  threshold: number,
  options?: { isVIP?: boolean }
) {
  validateAmount(amount);
  if (amount < threshold) return 0;
  return Math.floor(amount * getRate(options?.isVIP));
}

function validateAmount(amount: number): void {
  if (amount < 0) throw new Error('Amount must be positive');
}

function getRate(isVIP?: boolean): number {
  return isVIP ? DISCOUNT_RATES.vip : DISCOUNT_RATES.regular;
}
```

---

## Mock 外部依赖

### Mock 模块

```typescript
import { vi, describe, it, expect } from 'vitest';

// Mock 整个模块
vi.mock('../src/api', () => ({
  fetchUser: vi.fn().mockResolvedValue({ id: 1, name: 'Test' }),
}));

// 在单个测试中覆盖返回值
it('should handle api error', async () => {
  vi.mocked(fetchUser).mockRejectedValueOnce(new Error('Network error'));
  // ...
});
```

### Mock 定时器

```typescript
import { vi } from 'vitest';

describe('debounce', () => {
  beforeEach(() => { vi.useFakeTimers(); });
  afterEach(() => { vi.useRealTimers(); });

  it('should delay execution', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 100);

    debounced();
    expect(fn).not.toHaveBeenCalled();

    vi.advanceTimersByTime(100);
    expect(fn).toHaveBeenCalledOnce();
  });
});
```

### 测试异步代码

```typescript
// Promise
it('should fetch user', async () => {
  const user = await fetchUser('123');
  expect(user.name).toBe('Test');
});

// 拒绝的 Promise
it('should handle not found', async () => {
  await expect(fetchUser('999')).rejects.toThrow('User not found');
});

// callback
it('should call oncomplete', (done) => {
  processData(data, (result) => {
    expect(result.status).toBe('ok');
    done();
  });
});
```

---

## React 组件测试

```typescript
// 使用 @testing-library/react
import { render, screen, fireEvent } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect } from 'vitest';
import { Counter } from '../src/Counter';

describe('Counter', () => {
  it('should increment on click', async () => {
    const user = userEvent.setup();
    render(<Counter initialCount={0} />);

    expect(screen.getByText('Count: 0')).toBeInTheDocument();

    await user.click(screen.getByRole('button', { name: /increment/i }));
    expect(screen.getByText('Count: 1')).toBeInTheDocument();
  });
});
```

---

## 运行命令

```bash
# 运行所有测试
npx vitest

# 运行单个文件
npx vitest run tests/pricing.test.ts

# 监听模式
npx vitest --watch

# 覆盖率报告
npx vitest run --coverage

# 只运行匹配的测试
npx vitest run -t "calculateDiscount"
```
