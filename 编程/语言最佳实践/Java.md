# Java 最佳实践

## 项目结构

### 分层架构

```
src/main/java/com/example/project/
├── config/              # 配置类
├── controller/          # REST 控制器
├── service/             # 业务逻辑
│   └── impl/
├── repository/          # 数据访问层
├── model/               # 实体类
│   ├── entity/          # 数据库实体
│   ├── dto/             # 数据传输对象
│   └── vo/              # 视图对象
├── exception/           # 异常定义
├── util/                # 工具类
└── Application.java     # 启动类
```

## 基础编码

### 使用 Optional 处理空值

```java
// ❌ Bad - 返回 null
public User findUser(Long id) {
    return userRepository.findById(id); // 可能返回 null
}

// ✅ Good - 返回 Optional
public Optional<User> findUser(Long id) {
    return userRepository.findById(id);
}

// ✅ 链式调用
String email = findUser(id)
    .map(User::getEmail)
    .orElse("default@example.com");

// ✅ 提供替代值
User user = findUser(id)
    .orElseThrow(() -> new UserNotFoundException(id));

// ❌ 避免 isPresent + get
if (optional.isPresent()) {
    return optional.get().getName();
}

// ✅ 使用 map/filter/orElse
return optional.map(User::getName).orElse(null);
```

### 不可变对象

```java
// ✅ 使用 record（Java 16+）创建不可变数据载体
public record UserDto(
    Long id,
    String name,
    String email
) {}

// ✅ 使用 final 字段 + Builder 模式
public final class User {
    private final Long id;
    private final String name;
    private final String email;

    private User(Builder builder) {
        this.id = builder.id;
        this.name = builder.name;
        this.email = builder.email;
    }

    // Getter 方法，无 Setter
    public Long getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }

    public static Builder builder() {
        return new Builder();
    }

    public static class Builder {
        private Long id;
        private String name;
        private String email;

        public Builder id(Long id) { this.id = id; return this; }
        public Builder name(String name) { this.name = name; return this; }
        public Builder email(String email) { this.email = email; return this; }
        public User build() { return new User(this); }
    }
}
```

### 使用 var（Java 10+）

```java
// ✅ 右侧类型明确时使用 var
var users = new ArrayList<User>();     // 明显是 ArrayList<User>
var stream = users.stream();           // 明显是 Stream<User>
var map = Map.of("key", "value");      // 明显是 Map<String, String>

// ❌ 类型不明确时不使用 var
var result = process(data);            // process 返回什么？
var value = getService().calculate();  // 计算结果是什么类型？
```

### 使用 try-with-resources

```java
// ❌ Bad - 手动关闭
Connection conn = null;
try {
    conn = DriverManager.getConnection(url);
    // 使用 conn
} catch (SQLException e) {
    // 处理
} finally {
    if (conn != null) {
        try { conn.close(); } catch (SQLException e) { }
    }
}

// ✅ Good - 自动关闭
try (var conn = DriverManager.getConnection(url);
     var stmt = conn.prepareStatement(sql);
     var rs = stmt.executeQuery()) {
    // 使用资源
    while (rs.next()) {
        // 处理结果
    }
}
```

## 异常处理

### 异常层级

```java
// ✅ 基础业务异常
public abstract class BusinessException extends RuntimeException {
    private final String code;

    protected BusinessException(String code, String message) {
        super(message);
        this.code = code;
    }

    protected BusinessException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public String getCode() { return code; }
}

// ✅ 具体业务异常
public class NotFoundException extends BusinessException {
    public NotFoundException(String resource, Object id) {
        super("NOT_FOUND", resource + " " + id + " not found");
    }
}

public class ValidationException extends BusinessException {
    public ValidationException(String message) {
        super("VALIDATION_ERROR", message);
    }
}
```

### 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(NotFoundException e) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
                .body(new ErrorResponse(e.getCode(), e.getMessage()));
    }

    @ExceptionHandler(ValidationException.class)
    public ResponseEntity<ErrorResponse> handleValidation(ValidationException e) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
                .body(new ErrorResponse(e.getCode(), e.getMessage()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
        log.error("Unexpected error", e);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(new ErrorResponse("INTERNAL_ERROR", "Internal server error"));
    }
}
```

## Stream API

### 常用操作

```java
// ✅ 过滤 + 映射 + 收集
List<String> activeEmails = users.stream()
    .filter(User::isActive)
    .map(User::getEmail)
    .toList();  // Java 16+ 不可变列表

// ✅ 分组
Map<String, List<User>> byRole = users.stream()
    .collect(Collectors.groupingBy(User::getRole));

// ✅ 分区
Map<Boolean, List<User>> byAge = users.stream()
    .collect(Collectors.partitioningBy(u -> u.getAge() >= 18));

// ✅ 统计
IntSummaryStatistics stats = users.stream()
    .mapToInt(User::getAge)
    .summaryStatistics();

// ✅ 去重
List<User> unique = users.stream()
    .collect(Collectors.toMap(
        User::getEmail,
        u -> u,
        (existing, replacement) -> existing
    ))
    .values()
    .stream()
    .toList();

// ✅ 连接字符串
String names = users.stream()
    .map(User::getName)
    .collect(Collectors.joining(", "));
```

## 并发

### CompletableFuture

```java
// ✅ 异步组合
CompletableFuture<User> userFuture = fetchUserAsync(userId);
CompletableFuture<List<Order>> ordersFuture = fetchOrdersAsync(userId);

// 并行执行，合并结果
CompletableFuture<UserWithOrders> result = userFuture
    .thenCombine(ordersFuture, (user, orders) ->
        new UserWithOrders(user, orders)
    );

// ✅ 异常处理
CompletableFuture<String> safeFuture = fetchAsync(url)
    .exceptionally(ex -> {
        log.warn("Failed to fetch: {}", url, ex);
        return "fallback";
    });

// ✅ 超时
CompletableFuture<String> withTimeout = future
    .orTimeout(5, TimeUnit.SECONDS)
    .exceptionally(ex -> "timeout");
```

### 并发集合

```java
// ✅ 使用并发集合
ConcurrentHashMap<String, User> cache = new ConcurrentHashMap<>();
cache.putIfAbsent(key, user);
cache.computeIfAbsent(key, k -> loadUser(k));

// ✅ 不可变集合
List<String> list = List.of("a", "b", "c");
Map<String, Integer> map = Map.of("a", 1, "b", 2);
Set<String> set = Set.of("a", "b", "c");
```

## 设计模式

### 策略模式（函数式）

```java
// ✅ 使用枚举或函数式接口替代传统策略
public enum DiscountStrategy {
    REGULAR(price -> price),
    VIP(price -> price * 0.8),
    PREMIUM(price -> price * 0.7);

    private final DoubleFunction<Double> calculator;

    DiscountStrategy(DoubleFunction<Double> calculator) {
        this.calculator = calculator;
    }

    public double apply(double price) {
        return calculator.apply(price);
    }
}
```

### 工厂模式

```java
// ✅ 使用 Map 注册代替 if-else
public class NotificationFactory {
    private final Map<String, NotificationSender> senders;

    public NotificationFactory(List<NotificationSender> senderList) {
        this.senders = senderList.stream()
            .collect(Collectors.toMap(
                NotificationSender::getType,
                Function.identity()
            ));
    }

    public NotificationSender getSender(String type) {
        return Optional.ofNullable(senders.get(type))
            .orElseThrow(() -> new IllegalArgumentException("Unknown type: " + type));
    }
}
```

## 测试

### JUnit 5 最佳实践

```java
import static org.assertj.core.api.Assertions.*;

class UserServiceTest {

    @Test
    @DisplayName("创建用户 - 有效输入 - 返回创建的用户")
    void createUser_validInput_returnsCreatedUser() {
        // Arrange
        var request = new CreateUserRequest("Alice", "alice@test.com");
        when(repository.existsByEmail(request.email())).thenReturn(false);
        when(repository.save(any())).thenAnswer(inv -> inv.getArgument(0));

        // Act
        var result = service.createUser(request);

        // Assert
        assertThat(result.getName()).isEqualTo("Alice");
        assertThat(result.getEmail()).isEqualTo("alice@test.com");
        verify(repository).save(any());
    }

    // ✅ 参数化测试
    @ParameterizedTest
    @ValueSource(strings = {"", " ", "invalid-email", "@test.com"})
    @DisplayName("创建用户 - 无效邮箱 - 抛出异常")
    void createUser_invalidEmail_throwsException(String email) {
        var request = new CreateUserRequest("Alice", email);
        assertThatThrownBy(() -> service.createUser(request))
            .isInstanceOf(ValidationException.class);
    }
}
```

## 性能建议

- 使用 `StringBuilder` 拼接大量字符串
- 集合预分配大小 `new ArrayList<>(expectedSize)`
- 优先使用基本类型（`int`/`long`）而非包装类型（`Integer`/`Long`）
- 使用 `StringBuilder` 替代字符串 `+` 循环拼接
- 合理使用 `String.intern()`（注意 PermGen/Metaspace）
- 大文件读写使用 `BufferedReader`/`BufferedWriter`
- 使用对象池（如 `Apache Commons Pool`）管理昂贵资源

## 参考资源

- [Effective Java (Joshua Bloch)](https://www.oreilly.com/library/view/effective-java/9780134686097/)
- [Java 官方教程](https://docs.oracle.com/javase/tutorial/)
- [Baeldung](https://www.baeldung.com/)
- [Java Design Patterns](https://java-design-patterns.com/)
