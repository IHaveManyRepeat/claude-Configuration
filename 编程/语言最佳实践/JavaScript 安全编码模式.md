# JavaScript 安全编码模式

> 本文档是 [[安全概述]] 的语言特定补充，定义 JavaScript 项目开发中的安全编码规范。

## 输入验证

### 基础验证

```javascript
// ✅ Good - 验证用户输入
function validateUserInput(input) {
  const errors = [];
  
  if (!input || typeof input !== 'string') {
    errors.push('输入必须是字符串');
  }
  
  if (input.length < 2 || input.length > 100) {
    errors.push('长度必须在 2-100 之间');
  }
  
  // 白名单验证
  if (!/^[a-zA-Z0-9_]+$/.test(input)) {
    errors.push('只允许字母、数字、下划线');
  }
  
  return { valid: errors.length === 0, errors };
}

// ✅ Good - 使用正则验证邮箱
const isValidEmail = (email) => {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
};
```

### 防止命令注入

```javascript
// ❌ Bad - 使用用户输入执行命令
const { exec } = require('child_process');
exec(`ls -la ${userInput}`);  // 危险！

// ✅ Good - 避免直接执行用户输入
// 如果必须执行，使用白名单
const allowedCommands = ['status', 'info', 'stats'];
if (allowedCommands.includes(userInput)) {
  exec(`app ${userInput}`, callback);
}
```

## XSS 防护

### 输出转义

```javascript
// ✅ Good - HTML 转义
function escapeHtml(str) {
  const div = document.createElement('div');
  div.textContent = str;
  return div.innerHTML;
}

// 使用
const safeHtml = `<p>用户名: ${escapeHtml(userInput)}</p>`;
```

### 内容安全策略（CSP）

```html
<!-- ✅ Good - 在 HTML 中设置 CSP -->
<meta http-equiv="Content-Security-Policy" 
      content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'">
```

### DOM 安全

```javascript
// ✅ Good - 使用 textContent 而非 innerHTML
element.textContent = userInput;

// ✅ Good - 使用 innerText
element.innerText = userInput;

// ❌ Bad - 避免使用 innerHTML
element.innerHTML = userInput;  // 危险！

// ✅ Good - 创建元素而非字符串拼接
const p = document.createElement('p');
p.textContent = userInput;
container.appendChild(p);
```

## 敏感数据保护

```javascript
// ❌ Bad - 在前端存储敏感数据
localStorage.setItem('token', token);  // 危险！
sessionStorage.setItem('password', password);  // 危险！

// ✅ Good - 使用 HttpOnly Cookie
// 服务器端设置
res.cookie('session', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'strict'
});

// ✅ Good - 内存中存储（仅当前会话）
let sessionToken = null;  // 保存在内存中
```

## 依赖安全

```bash
# 定期审计
npm audit
npm audit fix

# 使用 snyk
npm install -g snyk
snyk test
```

## 安全检查清单

- [ ] 验证所有用户输入
- [ ] 避免直接执行用户输入（命令注入）
- [ ] 输出转义（XSS 防护）
- [ ] 使用 CSP 头部
- [ ] 使用 textContent 而非 innerHTML
- [ ] 敏感数据不存 localStorage/sessionStorage
- [ ] 定期运行 npm audit

## 参考资源

- [OWASP XSS 速查表](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [MDN CSP 指南](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CSP)