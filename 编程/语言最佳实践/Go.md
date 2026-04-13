# Go 最佳实践

## 项目结构

### 标准布局

```
project/
├── cmd/                 # 应用入口
│   └── server/
│       └── main.go
├── internal/            # 私有代码（不可被外部导入）
│   ├── handler/
│   ├── service/
│   ├── repository/
│   └── model/
├── pkg/                 # 可复用的公共库
│   └── utils/
├── api/                 # API 定义（proto, OpenAPI）
├── configs/             # 配置文件
├── go.mod
├── go.sum
└── Makefile
```

## 错误处理

### 显式处理每一个错误

```go
// ❌ Bad - 忽略错误
data, _ := os.ReadFile("config.json")

// ✅ Good - 始终处理错误
data, err := os.ReadFile("config.json")
if err != nil {
    return fmt.Errorf("reading config: %w", err)
}
```

### 自定义错误类型

```go
// ✅ 使用 fmt.Errorf 包装上下文
if err != nil {
    return fmt.Errorf("processing user %s: %w", userID, err)
}

// ✅ 自定义错误类型用于业务逻辑判断
type NotFoundError struct {
    Resource string
    ID       string
}

func (e *NotFoundError) Error() string {
    return fmt.Sprintf("%s %s not found", e.Resource, e.ID)
}

// ✅ 使用 errors.Is 和 errors.As
if errors.Is(err, os.ErrNotExist) {
    // 处理文件不存在
}

var notFound *NotFoundError
if errors.As(err, &notFound) {
    // 处理资源未找到
}
```

### sentinel errors

```go
// ✅ 预定义错误值
var (
    ErrNotFound    = errors.New("not found")
    ErrUnauthorized = errors.New("unauthorized")
    ErrBadRequest  = errors.New("bad request")
)
```

## 并发模式

### Goroutine 生命周期管理

```go
// ✅ 使用 context 控制取消
func (s *Server) Run(ctx context.Context) error {
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()

    // 使用 errgroup 管理多个 goroutine
    g, ctx := errgroup.WithContext(ctx)

    g.Go(func() error {
        return s.startHTTPServer(ctx)
    })

    g.Go(func() error {
        return s.startWorker(ctx)
    })

    // 等待任意一个 goroutine 返回错误
    if err := g.Wait(); err != nil {
        return fmt.Errorf("server stopped: %w", err)
    }
    return nil
}
```

### Channel 模式

```go
// ✅ 扇出/扇入模式
func fanOutFanIn(ctx context.Context, input <-chan int, workers int) <-chan int {
    // 扇出
    channels := make([]<-chan int, workers)
    for i := 0; i < workers; i++ {
        channels[i] = worker(ctx, input)
    }

    // 扇入
    merged := make(chan int)
    var wg sync.WaitGroup
    for _, ch := range channels {
        wg.Add(1)
        go func(c <-chan int) {
            defer wg.Done()
            for v := range c {
                select {
                case merged <- v:
                case <-ctx.Done():
                    return
                }
            }
        }(ch)
    }

    go func() {
        wg.Wait()
        close(merged)
    }()

    return merged
}
```

### 使用 sync 包

```go
// ✅ sync.Once 确保只执行一次
var once sync.Once
var instance *Service

func GetService() *Service {
    once.Do(func() {
        instance = &Service{}
    })
    return instance
}

// ✅ sync.Pool 对象复用
var bufPool = sync.Pool{
    New: func() any {
        return new(bytes.Buffer)
    },
}

func processData(data []byte) {
    buf := bufPool.Get().(*bytes.Buffer)
    defer func() {
        buf.Reset()
        bufPool.Put(buf)
    }()
    // 使用 buf...
}
```

## 接口设计

### 小接口原则

```go
// ✅ 小接口，由消费者定义
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

// 组合接口
type ReadWriter interface {
    Reader
    Writer
}

// ✅ 在使用方定义接口，而非实现方
// 消费者包
type UserGetter interface {
    GetUser(ctx context.Context, id string) (*User, error)
}

func NewHandler(getter UserGetter) *Handler {
    return &Handler{getter: getter}
}
```

### 避免返回接口

```go
// ❌ Bad - 返回接口
func NewService() Service { ... }

// ✅ Good - 返回具体类型，接受接口
func NewService(repo Repository) *ConcreteService { ... }
```

## 初始化与配置

### 函数选项模式

```go
// ✅ 函数选项模式
type ServerOption func(*Server)

func WithPort(port int) ServerOption {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout time.Duration) ServerOption {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func NewServer(opts ...ServerOption) *Server {
    s := &Server{
        port:    8080,        // 默认值
        timeout: 30 * time.Second,
    }
    for _, opt := range opts {
        opt(s)
    }
    return s
}
```

### 依赖注入

```go
// ✅ 构造函数注入
type UserService struct {
    repo      UserRepository
    cache     Cache
    logger    *slog.Logger
}

func NewUserService(
    repo UserRepository,
    cache Cache,
    logger *slog.Logger,
) *UserService {
    return &UserService{
        repo:   repo,
        cache:  cache,
        logger: logger,
    }
}
```

## 切片与 Map

### 切片最佳实践

```go
// ✅ 预分配已知大小的切片
nums := make([]int, 0, expectedSize)

// ✅ 切片拷贝
src := []int{1, 2, 3}
dst := make([]int, len(src))
copy(dst, src)

// ✅ 使用 maps 包（Go 1.21+）
import "maps"
cloned := maps.Clone(original)

// ✅ 使用 slices 包（Go 1.21+）
import "slices"
slices.Sort(nums)
idx := slices.Index(nums, target)
```

## 测试

### 表驱动测试

```go
func TestAdd(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 1, 2, 3},
        {"negative numbers", -1, -2, -3},
        {"zero", 0, 0, 0},
        {"mixed", -1, 1, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            got := Add(tt.a, tt.b)
            assert.Equal(t, tt.expected, got)
        })
    }
}
```

## 性能建议

- 使用 `strings.Builder` 拼接字符串
- 预分配 slice 和 map（`make([]T, 0, n)`）
- 避免 `append` 后的切片引用泄漏
- 使用 `sync.Pool` 复用临时对象
- 使用 `pprof` 进行性能分析
- I/O 密集用 goroutine，CPU 密集控制并发数（`runtime.GOMAXPROCS`）

## 参考资源

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Go by Example](https://gobyexample.com/)
- [100 Go Mistakes](https://100go.co/)
