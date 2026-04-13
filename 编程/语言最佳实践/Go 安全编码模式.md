# Go 安全编码模式

> 本文档是 [[安全概述]] 的语言特定补充，定义 Go 项目开发中的安全编码规范。

## 输入验证

### 参数化查询（防止 SQL 注入）

```go
// ✅ Good - 参数化查询
func GetUserByID(db *sql.DB, id string) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE id = $1"
    row := db.QueryRow(query, id)
    
    var user User
    err := row.Scan(&user.ID, &user.Name, &user.Email)
    if err == sql.ErrNoRows {
        return nil, nil
    }
    return &user, err
}

// ❌ Bad - 字符串拼接（SQL 注入风险）
func GetUserByID(db *sql.DB, id string) (*User, error) {
    query := "SELECT id, name, email FROM users WHERE id = '" + id + "'" // 危险！
}
```

### 模板转义（防止 XSS）

```go
// ✅ Good - html/template 自动转义
import "html/template"

func renderPage(w http.ResponseWriter, r *http.Request) {
    tmpl := template.Must(template.ParseFiles("template.html"))
    // 用户输入会被自动转义
    tmpl.Execute(w, map[string]string{
        "username": r.FormValue("username"), // 自动转义
    })
}

// ❌ Bad - text/template 不转义
import "text/template" // 危险！
```

## 密码存储

```go
import "golang.org/x/crypto/bcrypt"

// ✅ Good - 使用 bcrypt 哈希密码
func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}

func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

## 密钥管理

```go
import "os"

// ✅ Good - 从环境变量读取
func GetAPIKey() string {
    key := os.Getenv("API_KEY")
    if key == "" {
        panic("API_KEY environment variable is required")
    }
    return key
}

// ✅ Good - 使用配置结构
type Config struct {
    DBPassword string `required:"true" split_words:"false"`
    APIKey     string `required:"true"`
}

// ❌ Bad - 硬编码密钥
const API_KEY = "sk-1234567890abcdef" // 危险！
```

## 认证与会话

```go
import (
    "crypto/rand"
    "encoding/base64"
)

// ✅ Good - 安全令牌生成
func GenerateToken() (string, error) {
    b := make([]byte, 32)
    _, err := rand.Read(b)
    if err != nil {
        return "", err
    }
    return base64.URLEncoding.EncodeToString(b), nil
}

// ✅ Good - 使用 http.SameSite
cookie := http.Cookie{
    Name:     "session",
    Value:    sessionID,
    Secure:   true,
    HttpOnly: true,
    SameSite: http.SameSiteStrictMode,
}
```

## 并发安全

```go
import "sync"

// ✅ Good - 互斥锁保护共享资源
type SafeCounter struct {
    mu    sync.Mutex
    count int
}

func (c *SafeCounter) Increment() {
    c.mu.Lock()
    defer c.mu.Unlock()
    c.count++
}

// ✅ Good - 使用 sync.Map 并发场景
var cache sync.Map

func SetCache(key, value string) {
    cache.Store(key, value)
}

func GetCache(key string) (string, bool) {
    val, ok := cache.Load(key)
    if !ok {
        return "", false
    }
    return val.(string), true
}
```

## 错误处理

```go
// ✅ Good - 不泄露敏感信息
func handleError(w http.ResponseWriter, err error) {
    // 记录详细错误用于调试
    log.Error("Internal error", "error", err)
    
    // 对用户返回通用消息
    http.Error(w, "Internal server error", http.StatusInternalServerError)
}

// ❌ Bad - 泄露堆栈信息
func handleError(w http.ResponseWriter, err error) {
    http.Error(w, err.Error(), http.StatusInternalServerError) // 危险！
}
```

## 依赖安全

```bash
# 定期运行安全审计
go list -json all | jq -r '.Path + "@" + .Version' | go run -mod=mod honnef.co/go/tools/cmd/gosec@latest -

# 使用 govulncheck 检查漏洞
go install golang.org/x/vuln/cmd/govulncheck@latest
govulncheck ./...
```

## 安全检查清单

- [ ] 使用参数化查询，防止 SQL 注入
- [ ] 使用 html/template 渲染用户输入
- [ ] 密码使用 bcrypt 存储
- [ ] 密钥从环境变量读取，无硬编码
- [ ] Cookie 设置 HttpOnly、Secure、SameSite
- [ ] 并发访问共享资源使用 mutex
- [ ] 错误消息不泄露系统细节
- [ ] 定期运行 go vet、gosec、govulncheck

## 参考资源

- [Go 安全编码指南](https://golang.org/doc/security)
- [OWASP Go 速查表](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)