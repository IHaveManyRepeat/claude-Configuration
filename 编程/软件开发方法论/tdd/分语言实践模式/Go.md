# TDD 实践模式 — Go

#编程语言/Go #软件开发方法论/TDD

---

## 工具链

| 工具 | 用途 | 安装 |
|------|------|------|
| `testing` | 标准测试包（内置） | 无需安装 |
| testify | 断言和 Mock 增强 | `go get github.com/stretchr/testify` |
| gomock | 接口 Mock 生成 | `go install go.uber.org/mock/mockgen@latest` |
| `go test` | 运行测试（内置） | 无需安装 |

---

## 纯函数 TDD 示例

### RED — 写测试

```go
// pricing_test.go
package pricing

import (
    "testing"
    "github.com/stretchr/testify/assert"
)

func TestCalculateDiscount(t *testing.T) {
    t.Run("returns zero below threshold", func(t *testing.T) {
        result := CalculateDiscount(50, 100, false)
        assert.Equal(t, 0, result)
    })

    t.Run("returns zero at threshold", func(t *testing.T) {
        result := CalculateDiscount(100, 100, false)
        assert.Equal(t, 0, result)
    })

    t.Run("applies regular rate above threshold", func(t *testing.T) {
        result := CalculateDiscount(200, 100, false)
        assert.Equal(t, 10, result)
    })

    t.Run("applies VIP rate", func(t *testing.T) {
        result := CalculateDiscount(200, 100, true)
        assert.Equal(t, 20, result)
    })

    t.Run("panics on negative amount", func(t *testing.T) {
        assert.Panics(t, func() {
            CalculateDiscount(-1, 100, false)
        })
    })
}
```

### GREEN — 最简实现

```go
// pricing.go
package pricing

const (
    RateVIP     = 0.10
    RateRegular = 0.05
)

func CalculateDiscount(amount, threshold float64, isVIP bool) int {
    if amount < 0 {
        panic("amount must be positive")
    }
    if amount < threshold {
        return 0
    }
    rate := RateRegular
    if isVIP {
        rate = RateVIP
    }
    return int(amount * rate)
}
```

### REFACTOR — 优化

```go
// pricing.go
package pricing

type DiscountConfig struct {
    Threshold float64
    VIPRate   float64
    RegularRate float64
}

var DefaultConfig = DiscountConfig{
    Threshold:   100,
    VIPRate:     0.10,
    RegularRate: 0.05,
}

func CalculateDiscount(amount float64, cfg DiscountConfig, isVIP bool) (int, error) {
    if amount < 0 {
        return 0, fmt.Errorf("amount must be positive, got %f", amount)
    }
    if amount < cfg.Threshold {
        return 0, nil
    }
    rate := cfg.RegularRate
    if isVIP {
        rate = cfg.VIPRate
    }
    return int(amount * rate), nil
}
```

---

## 接口 Mock（Go 核心模式）

Go 通过**接口**实现解耦和 Mock，是 Go TDD 的核心技巧。

### 定义接口

```go
// user_repository.go
package user

type UserRepository interface {
    FindByID(id string) (*User, error)
    Save(user *User) error
}
```

### 手写 Mock（简单场景）

```go
// mocks/user_repository.go
package mocks

import "your-project/user"

type MockUserRepo struct {
    FindByIDFunc func(id string) (*user.User, error)
    SaveFunc     func(user *user.User) error
}

func (m *MockUserRepo) FindByID(id string) (*user.User, error) {
    return m.FindByIDFunc(id)
}

func (m *MockUserRepo) Save(user *user.User) error {
    return m.SaveFunc(user)
}
```

### 使用 Mock 测试

```go
// user_service_test.go
package user

import (
    "errors"
    "testing"
    "your-project/mocks"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestGetUser_Success(t *testing.T) {
    // Arrange
    repo := &mocks.MockUserRepo{
        FindByIDFunc: func(id string) (*User, error) {
            return &User{ID: id, Name: "Test"}, nil
        },
    }
    svc := NewUserService(repo)

    // Act
    user, err := svc.GetUser("123")

    // Assert
    require.NoError(t, err)
    assert.Equal(t, "Test", user.Name)
}

func TestGetUser_NotFound(t *testing.T) {
    repo := &mocks.MockUserRepo{
        FindByIDFunc: func(id string) (*User, error) {
            return nil, ErrNotFound
        },
    }
    svc := NewUserService(repo)

    _, err := svc.GetUser("999")
    assert.True(t, errors.Is(err, ErrNotFound))
}
```

### 用 gomock 生成 Mock（复杂接口）

```bash
# 在接口文件上生成 mock
mockgen -source=user_repository.go -destination=mocks/user_repository.go
```

```go
func TestWithGeneratedMock(t *testing.T) {
    ctrl := gomock.NewController(t)
    defer ctrl.Finish()

    repo := mock_user.NewMockUserRepository(ctrl)
    repo.EXPECT().FindByID("123").Return(&User{Name: "Test"}, nil)

    svc := NewUserService(repo)
    user, err := svc.GetUser("123")

    require.NoError(t, err)
    assert.Equal(t, "Test", user.Name)
}
```

---

## 表驱动测试（Go 惯用模式）

```go
func TestCalculateDiscount_Table(t *testing.T) {
    tests := []struct {
        name      string
        amount    float64
        threshold float64
        isVIP     bool
        want      int
        wantPanic bool
    }{
        {"below threshold", 50, 100, false, 0, false},
        {"at threshold", 100, 100, false, 0, false},
        {"regular above", 200, 100, false, 10, false},
        {"VIP above", 200, 100, true, 20, false},
        {"negative amount", -1, 100, false, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            if tt.wantPanic {
                assert.Panics(t, func() {
                    CalculateDiscount(tt.amount, tt.threshold, tt.isVIP)
                })
                return
            }
            got := CalculateDiscount(tt.amount, tt.threshold, tt.isVIP)
            assert.Equal(t, tt.want, got)
        })
    }
}
```

---

## 测试 HTTP Handler

```go
import "net/http/httptest"

func TestHandleGetUser(t *testing.T) {
    req := httptest.NewRequest("GET", "/users/123", nil)
    rec := httptest.NewRecorder()

    handler := HandleGetUser(mockUserService())
    handler.ServeHTTP(rec, req)

    assert.Equal(t, http.StatusOK, rec.Code)

    var resp UserResponse
    json.Unmarshal(rec.Body.Bytes(), &resp)
    assert.Equal(t, "Test", resp.Name)
}
```

---

## 运行命令

```bash
# 运行当前包的测试
go test

# 运行所有包
go test ./...

# 显示详细输出
go test -v ./...

# 运行匹配的测试
go test -run TestCalculateDiscount ./...

# 覆盖率
go test -cover ./...
go test -coverprofile=coverage.out ./... && go tool cover -html=coverage.out

# 竞态检测
go test -race ./...

# 运行基准测试
go test -bench=. -benchmem ./...
```
