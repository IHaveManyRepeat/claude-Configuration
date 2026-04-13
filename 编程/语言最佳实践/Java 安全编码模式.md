# Java 安全编码模式

> 本文档是 [[安全概述]] 的语言特定补充，定义 Java 项目开发中的安全编码规范。

## 输入验证

### 使用 Bean Validation

```java
import jakarta.validation.constraints.*;
import jakarta.validation.Valid;

public class UserCreateRequest {
    @NotBlank(message = "名称不能为空")
    @Size(min = 2, max = 100, message = "名称长度必须在 2-100 之间")
    private String name;

    @NotBlank
    @Email(message = "邮箱格式不正确")
    private String email;

    @Min(value = 0, message = "年龄不能为负数")
    @Max(value = 150, message = "年龄不能超过 150")
    private Integer age;

    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "只允许字母、数字、下划线")
    private String username;
}

// 验证
@RestController
public class UserController {
    @PostMapping("/users")
    public ResponseEntity<?> createUser(@Valid @RequestBody UserCreateRequest request) {
        // 验证失败会自动返回 400
        return ResponseEntity.ok(userService.create(request));
    }
}
```

## SQL 注入防护

### 使用 PreparedStatement

```java
// ✅ Good - 参数化查询
public Optional<User> findById(Connection conn, String id) throws SQLException {
    String sql = "SELECT id, name, email FROM users WHERE id = ?";
    
    try (PreparedStatement stmt = conn.prepareStatement(sql)) {
        stmt.setString(1, id);
        
        try (ResultSet rs = stmt.executeQuery()) {
            if (rs.next()) {
                return Optional.of(new User(
                    rs.getString("id"),
                    rs.getString("name"),
                    rs.getString("email")
                ));
            }
        }
    }
    return Optional.empty();
}

// ❌ Bad - 字符串拼接（SQL 注入风险）
public User findById(Connection conn, String id) throws SQLException {
    String sql = "SELECT id, name, email FROM users WHERE id = '" + id + "'";  // 危险！
    // ...
}
```

### 使用 JPA/ORM

```java
// ✅ Good - 使用 JPA
public interface UserRepository extends JpaRepository<User, String> {
    Optional<User> findByEmail(String email);
    
    @Query("SELECT u FROM User u WHERE u.name = :name")
    List<User> findByName(@Param("name") String name);
}
```

## 密码存储

```java
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;

// ✅ Good - 使用 BCrypt 哈希密码
@Service
public class PasswordService {
    private final BCryptPasswordEncoder encoder = new BCryptPasswordEncoder();
    
    public String encodePassword(String rawPassword) {
        return encoder.encode(rawPassword);
    }
    
    public boolean matches(String rawPassword, String encodedPassword) {
        return encoder.matches(rawPassword, encodedPassword);
    }
}
```

## XSS 防护

### 使用 Spring Security

```java
@Configuration
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .xssProtection(xss -> xss.enable())
                .contentSecurityPolicy(csp -> css
                    .policyDirectives("default-src 'self'; script-src 'self'")
                )
            );
        return http.build();
    }
}
```

### 输出转义

```java
// ✅ Good - 使用 Apache Commons Text 转义
import org.apache.commons.text.StringEscapeUtils;

public class SafeHtmlUtil {
    public static String escapeHtml(String input) {
        return StringEscapeUtils.escapeHtml4(input);
    }
    
    public static String escapeJavaScript(String input) {
        return StringEscapeUtils.escapeEcmaScript(input);
    }
}
```

## 序列化安全

```java
import java.io.*;

// ❌ Bad - 反序列化不可信数据
public Object deserialize(byte[] data) throws Exception {
    ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(data));
    return ois.readObject();  // 危险！
}

// ✅ Good - 使用白名单
public class SafeObjectInputStream extends ObjectInputStream {
    private final Set<String> allowedClasses = Set.of(
        "com.example.MyData",
        "java.util.ArrayList",
        "java.util.HashMap"
    );
    
    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
        String name = desc.getName();
        if (!allowedClasses.contains(name)) {
            throw new InvalidClassException("Unauthorized class: " + name);
        }
        return super.resolveClass(desc);
    }
}
```

## 敏感数据保护

```java
// ✅ Good - 脱敏返回数据
public class UserResponse {
    private String id;
    private String name;
    private String email;
    
    // Getters
    
    public String getEmail() {
        return maskEmail(email);
    }
    
    private String maskEmail(String email) {
        if (email == null) return null;
        int atIndex = email.indexOf('@');
        if (atIndex <= 1) return email;
        return email.charAt(0) + "***" + email.charAt(atIndex - 1) + email.substring(atIndex);
    }
}
```

## 依赖安全

```bash
# 使用 OWASP Dependency Check
./mvnw org.owasp:dependency-check-maven:check

# 使用 Snyk
mvn snyk:snyk
```

## 安全检查清单

- [ ] 使用 Bean Validation 验证输入
- [ ] 使用 PreparedStatement 参数化查询
- [ ] 密码使用 BCrypt 存储
- [ ] 配置 Spring Security XSS 防护
- [ ] 序列化使用白名单
- [ ] 敏感数据脱敏返回
- [ ] 定期运行 dependency-check

## 参考资源

- [OWASP Java 速查表](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [Spring Security 文档](https://spring.io/projects/spring-security)