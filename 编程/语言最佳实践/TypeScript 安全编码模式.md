# TypeScript 安全编码模式

> 本文档是 [[安全概述]] 的语言特定补充，定义 TypeScript 项目开发中的安全编码规范。

## 类型安全作为安全屏障

### 严格类型检查

```typescript
// ✅ Good - 启用 strict 模式
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true
  }
}

// ✅ Good - 使用 unknown 处理外部数据
function processInput(data: unknown): User {
  if (!isValidUser(data)) {
    throw new Error('Invalid input');
  }
  return data as User;
}

// ✅ Good - 类型守卫
function isValidUser(data: unknown): data is User {
  return (
    typeof data === 'object' &&
    data !== null &&
    'id' in data &&
    'name' in data &&
    typeof (data as User).id === 'string' &&
    typeof (data as User).name === 'string'
  );
}
```

## 输入验证

### 使用 Zod 验证

```typescript
import { z } from 'zod';

// ✅ Good - 使用 Zod 定义验证模式
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(150),
  role: z.enum(['admin', 'user', 'guest']),
});

type User = z.infer<typeof UserSchema>;

// 验证函数
function validateUser(data: unknown): User {
  return UserSchema.parse(data);
}

// 验证并返回结果
function safeValidateUser(data: unknown) {
  const result = UserSchema.safeParse(data);
  if (!result.success) {
    return { success: false, errors: result.error.issues };
  }
  return { success: true, data: result.data };
}
```

### 使用 class-validator（Angular/NestJS）

```typescript
import { IsString, IsEmail, MinLength, MaxLength, IsEnum, ValidateIf } from 'class-validator';

export class CreateUserDto {
  @IsString()
  @MinLength(1)
  @MaxLength(100)
  name!: string;

  @IsEmail()
  email!: string;

  @IsEnum(['admin', 'user', 'guest'])
  role!: 'admin' | 'user' | 'guest';
}
```

## XSS 防护

### 输出转义

```typescript
// ✅ Good - HTML 转义
function escapeHtml(str: string): string {
  const map: Record<string, string> = {
    '&': '&amp;',
    '<': '&lt;',
    '>': '&gt;',
    '"': '&quot;',
    "'": '&#x27;',
    '/': '&#x2F;',
  };
  return str.replace(/[&<>"'/]/g, (char) => map[char]);
}

// ✅ Good - React 中使用
const UserDisplay: React.FC<{ name: string }> = ({ name }) => (
  <div>{/* React 默认转义 */}</div>
);

// ❌ Bad - 避免 dangerouslySetInnerHTML
<div dangerouslySetInnerHTML={{ __html: userInput }} />  // 危险！
```

### 内容安全策略

```typescript
// ✅ Good - 响应头设置（Express）
import helmet from 'helmet';

app.use(helmet.contentSecurityPolicy({
  directives: {
    defaultSrc: ["'self'"],
    scriptSrc: ["'self'"],
    styleSrc: ["'self'", "'unsafe-inline'"],
    imgSrc: ["'self'", 'data:', 'https:'],
  },
}));
```

## API 安全

### 请求验证

```typescript
// ✅ Good - 使用中间件验证
import { z } from 'zod';

const requestSchema = z.object({
  params: z.object({
    id: z.string().uuid(),
  }),
  body: z.object({
    name: z.string().min(1),
    email: z.string().email(),
  }),
});

function validateRequest<T>(schema: z.ZodSchema<T>, data: unknown): T {
  return schema.parse(data);
}
```

### 速率限制

```typescript
import rateLimit from 'express-rate-limit';

// ✅ Good - 速率限制
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 分钟
  max: 100, // 每个 IP 最多 100 次请求
  message: 'Too many requests',
});
```

## 敏感数据处理

```typescript
// ✅ Good - 脱敏返回数据
interface User {
  id: string;
  name: string;
  email: string;
  passwordHash: string;
}

function sanitizeUser(user: User): SanitizedUser {
  const { passwordHash, ...rest } = user;
  return {
    ...rest,
    email: maskEmail(user.email),
  };
}

function maskEmail(email: string): string {
  const [local, domain] = email.split('@');
  const maskedLocal = local.charAt(0) + '***' + local.charAt(local.length - 1);
  return `${maskedLocal}@${domain}`;
}
```

## 依赖安全

```bash
# 定期审计
npm audit
npm audit fix

# 使用 snyk
npx snyk test
```

## 安全检查清单

- [ ] 启用 TypeScript strict 模式
- [ ] 使用 Zod/class-validator 验证输入
- [ ] 使用 unknown 处理外部数据
- [ ] 输出转义（XSS 防护）
- [ ] 设置 CSP 响应头
- [ ] 实现速率限制
- [ ] 敏感数据脱敏返回
- [ ] 定期运行 npm audit

## 参考资源

- [OWASP TypeScript 速查表](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [Zod 文档](https://zod.dev/)
- [Helmet.js](https://helmetjs.github.io/)