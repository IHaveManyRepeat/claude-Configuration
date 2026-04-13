# TDD 实践模式 — Java

#编程语言/Java #软件开发方法论/TDD

---

## 工具链

| 工具 | 用途 | 配置方式 |
|------|------|---------|
| JUnit 5 | 测试框架 | Maven: `junit-jupiter` / Gradle: `testImplementation` |
| AssertJ | 流式断言 | Maven: `assertj-core` / Gradle: `testImplementation` |
| Mockito | Mock 框架 | Maven: `mockito-core` / Gradle: `testImplementation` |
| JaCoCo | 覆盖率 | Maven/Gradle 插件 |

### Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter</artifactId>
        <version>5.10.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>3.25.3</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>5.11.0</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle 依赖

```groovy
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter:5.10.2'
    testImplementation 'org.assertj:assertj-core:3.25.3'
    testImplementation 'org.mockito:mockito-core:5.11.0'
}

test {
    useJUnitPlatform()
}
```

---

## 纯函数 TDD 示例

### RED — 写测试

```java
// src/test/java/com/example/pricing/PricingServiceTest.java
package com.example.pricing;

import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class PricingServiceTest {

    @Nested
    @DisplayName("calculateDiscount")
    class CalculateDiscount {

        @Test
        @DisplayName("低于阈值时返回 0")
        void returnsZeroBelowThreshold() {
            int result = PricingService.calculateDiscount(50, 100, false);
            assertThat(result).isZero();
        }

        @Test
        @DisplayName("VIP 用户享受 10% 折扣")
        void appliesVipDiscount() {
            int result = PricingService.calculateDiscount(200, 100, true);
            assertThat(result).isEqualTo(20);
        }

        @Test
        @DisplayName("负数金额抛出异常")
        void throwsOnNegativeAmount() {
            assertThatThrownBy(() -> PricingService.calculateDiscount(-1, 100, false))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessage("Amount must be positive");
        }
    }
}
```

### GREEN — 最简实现

```java
// src/main/java/com/example/pricing/PricingService.java
package com.example.pricing;

public class PricingService {

    public static int calculateDiscount(double amount, double threshold, boolean isVIP) {
        if (amount < 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        if (amount < threshold) {
            return 0;
        }
        double rate = isVIP ? 0.10 : 0.05;
        return (int) Math.floor(amount * rate);
    }
}
```

### REFACTOR — 优化

```java
public class PricingService {

    private static final double VIP_RATE = 0.10;
    private static final double REGULAR_RATE = 0.05;

    public static int calculateDiscount(double amount, double threshold, boolean isVIP) {
        validateAmount(amount);
        if (amount < threshold) {
            return 0;
        }
        return (int) Math.floor(amount * getRate(isVIP));
    }

    private static void validateAmount(double amount) {
        if (amount < 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
    }

    private static double getRate(boolean isVIP) {
        return isVIP ? VIP_RATE : REGULAR_RATE;
    }
}
```

---

## Mockito Mock 模式

### Mock 依赖注入

```java
// src/test/java/com/example/user/UserServiceTest.java
package com.example.user;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("根据 ID 查找用户成功")
    void findUserById_Success() {
        // Arrange
        User expected = new User("123", "Test");
        when(userRepository.findById("123")).thenReturn(Optional.of(expected));

        // Act
        User result = userService.getUser("123");

        // Assert
        assertThat(result).isEqualTo(expected);
        verify(userRepository).findById("123");
        verifyNoMoreInteractions(userRepository);
    }

    @Test
    @DisplayName("用户不存在时抛出异常")
    void findUserById_NotFound() {
        when(userRepository.findById("999")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> userService.getUser("999"))
            .isInstanceOf(UserNotFoundException.class);

        verify(userRepository).findById("999");
    }
}
```

### 行为验证 vs 状态验证

```java
// 状态验证（推荐）：验证返回值
@Test
void stateVerification() {
    when(repository.count()).thenReturn(5L);
    assertThat(service.getTotalCount()).isEqualTo(5L);
}

// 行为验证：验证是否被调用
@Test
void behaviorVerification() {
    service.createUser("Test");
    verify(repository).save(argThat(user ->
        user.getName().equals("Test")
    ));
}

// 验证调用次数
verify(repository, times(1)).save(any());
verify(repository, never()).delete(any());
```

### 参数化测试

```java
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import org.junit.jupiter.params.provider.ValueSource;

class PricingServiceParameterizedTest {

    @ParameterizedTest(name = "amount={0}, threshold=100, expected={1}")
    @CsvSource({
        "50,  0",
        "100, 0",
        "200, 10"
    })
    void calculateDiscount_variousInputs(double amount, int expected) {
        assertThat(PricingService.calculateDiscount(amount, 100, false))
            .isEqualTo(expected);
    }

    @ParameterizedTest
    @ValueSource(doubles = {-1, -100, -0.01})
    void calculateDiscount_negativeAmountsThrow(double amount) {
        assertThatThrownBy(() -> PricingService.calculateDiscount(amount, 100, false))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

---

## Spring Boot 集成测试

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.beans.factory.annotation.Autowired;

@SpringBootTest
class OrderServiceIntegrationTest {

    @MockBean
    private PaymentGateway paymentGateway;

    @Autowired
    private OrderService orderService;

    @Test
    void placeOrder_successfulPayment() {
        when(paymentGateway.charge(any())).thenReturn(true);

        OrderResult result = orderService.placeOrder(new OrderRequest("item-1", 100));

        assertThat(result.isSuccess()).isTrue();
    }
}
```

---

## 运行命令

```bash
# Maven
mvn test                                    # 运行所有测试
mvn test -Dtest=PricingServiceTest          # 运行单个类
mvn test -Dtest="PricingServiceTest#*"      # 运行类中所有测试
mvn test -Dtest="*calculateDiscount*"       # 按方法名匹配

# Gradle
./gradlew test                              # 运行所有测试
./gradlew test --tests PricingServiceTest   # 运行单个类
./gradlew test --tests "*.calculateDiscount*"  # 按方法名匹配

# 覆盖率（JaCoCo）
mvn jacoco:report                           # 生成覆盖率报告
./gradlew jacocoTestReport                  # Gradle 生成覆盖率报告
```
