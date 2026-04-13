# TypeScript 最佳实践

## 类型系统

### 优先使用 `interface` 定义对象类型

```typescript
// ✅ Good - interface 可扩展、可合并
interface User {
  id: string;
  name: string;
  email: string;
}

// ✅ 简单联合类型用 type
type Status = "active" | "inactive" | "suspended";
type Result<T> = { success: true; data: T } | { success: false; error: string };
```

### 严格模式必备

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

### 避免类型断言，使用类型守卫

```typescript
// ❌ Bad
const user = data as User;

// ✅ Good - 类型守卫
function isUser(data: unknown): data is User {
  return (
    typeof data === "object" &&
    data !== null &&
    "id" in data &&
    "name" in data
  );
}

if (isUser(data)) {
  // data 安全使用
}
```

### 使用 `satisfies` 操作符进行类型校验而不拓宽类型

```typescript
type Colors = Record<string, [number, number, number] | string>;

// ✅ satisfies 校验类型但保留字面量类型
const palette = {
  red: [255, 0, 0],
  green: "#00ff00",
  blue: [0, 0, 255],
} satisfies Colors;

// palette.green 的类型是 "#00ff00" 而非 string
palette.green.toUpperCase(); // OK
```

## 日常模式

### 使用 `readonly` 和 `as const` 保证不可变性

```typescript
// ✅ 配置对象用 as const
const ROUTES = {
  home: "/",
  about: "/about",
  contact: "/contact",
} as const;

// ✅ 函数参数用 readonly
function processItems(items: readonly string[]): void {
  // items.push() // 编译错误
}
```

### Discriminated Union 处理多状态

```typescript
// ✅ 标签联合类型
type LoadingState = { status: "loading" };
type SuccessState = { status: "success"; data: User[] };
type ErrorState = { status: "error"; message: string };

type State = LoadingState | SuccessState | ErrorState;

function render(state: State) {
  switch (state.status) {
    case "loading":
      return <Spinner />;
    case "success":
      return <UserList users={state.data} />;
    case "error":
      return <Error message={state.message} />;
  }
}
```

### 泛型约束

```typescript
// ✅ 约束泛型确保类型安全
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

// ✅ 使用泛型推断而非手动标注
function createMap<T>(items: T[], keyFn: (item: T) => string): Map<string, T> {
  return new Map(items.map((item) => [keyFn(item), item]));
}
```

### 使用工具类型

```typescript
// ✅ 内置工具类型
type PartialUser = Partial<User>;           // 所有字段可选
type RequiredUser = Required<User>;         // 所有字段必填
type PickUser = Pick<User, "id" | "name">;  // 选取字段
type OmitUser = Omit<User, "email">;        // 排除字段
type ReadonlyUser = Readonly<User>;         // 所有字段只读

// ✅ 自定义工具类型
type DeepPartial<T> = { [P in keyof T]?: DeepPartial<T[P]> };
type Nullable<T> = T | null;
type Optional<T, K extends keyof T> = Omit<T, K> & Partial<Pick<T, K>>;
```

## 错误处理

```typescript
// ✅ Result 模式替代 try-catch
type Result<T, E = Error> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function divide(a: number, b: number): Result<number> {
  if (b === 0) return { ok: false, error: new Error("Division by zero") };
  return { ok: true, value: a / b };
}

const result = divide(10, 2);
if (result.ok) {
  console.log(result.value);
} else {
  console.error(result.error);
}
```

## 项目组织

### 模块化结构

```
src/
├── types/           # 共享类型定义
│   ├── index.ts
│   └── api.ts
├── utils/           # 纯工具函数
├── services/        # 业务逻辑
├── components/      # UI 组件
└── __tests__/       # 测试文件
```

### 使用 barrel exports

```typescript
// types/index.ts
export type { User, UserRole } from "./user";
export type { ApiResponse, PaginatedResponse } from "./api";
```

### 枚举最佳实践

```typescript
// ❌ Bad - 数字枚举有反向映射问题
enum Direction {
  Up,    // 0
  Down,  // 1
}

// ✅ Good - 字符串枚举
enum Direction {
  Up = "UP",
  Down = "DOWN",
}

// ✅ Better - as const 对象（tree-shakable）
const Direction = {
  Up: "UP",
  Down: "DOWN",
} as const;
type Direction = (typeof Direction)[keyof typeof Direction];
```

## 性能建议

- 使用 `tsconfig` 的 `references` 实现项目引用，提升大型项目编译速度
- 避免过度使用 `any`，使用 `unknown` 替代
- 避免在热路径中使用复杂的条件类型
- 使用 `isolatedModules: true` 确保每个文件可独立转译

## 参考资源

- [TypeScript 官方文档](https://www.typescriptlang.org/docs/)
- [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/)
- [Type Challenges](https://github.com/type-challenges/type-challenges)
