
| 通知类型     | 注解                | 执行时机                               | 特点与用途                                    |
| -------- | ----------------- | ---------------------------------- | ---------------------------------------- |
| **前置通知** | `@Before`         | 在**目标方法执行之前**执行。                   | 适用于权限校验、日志记录、参数检查等。                      |
| **后置通知** | `@AfterReturning` | 在**目标方法正常执行完毕后**执行（未发生异常）。         | 适用于记录操作结果、更新日志等。                         |
| **环绕通知** | `@Around`         | **包围**目标方法，可以在方法执行前和执行后自定义行为。      | 功能最强大，可以控制是否执行目标方法、修改返回值等。适用于事务管理、性能监控等。 |
| **异常通知** | `@AfterThrowing`  | 在**目标方法抛出异常后**执行。                  | 适用于异常处理、事务回滚、异常报警等。                      |
| **最终通知** | `@After`          | 在**目标方法执行完毕后**执行，**无论是否发生异常**都会执行。 | 类似于 `finally`代码块，适用于资源清理、解锁等操作。          |


1. **通知选择**：
    
    - **权限/校验**：使用 `@Before`
        
    - **日志记录**：使用 `@AfterReturning`或 `@Around`
        
    - **性能监控**：使用 `@Around`
        
    - **异常处理**：使用 `@AfterThrowing`
        
    - **资源清理**：使用 `@After`
        
    
2. **执行顺序**：如图表所示，理解不同通知的执行时机至关重要。
    
3. **最佳实践**：
    
    - **重用切点**：使用 `@Pointcut`避免重复代码。
        
    - **获取上下文**：通过 `JoinPoint`获取方法信息。
        
    - **控制顺序**：使用 `@Order`管理多切面执行顺序。
        
### 一、 切面类基础框架

首先，我们创建一个切面类，定义可重用的切点（对应图3），并演示如何获取连接点信息（对应图4）。

```
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.springframework.stereotype.Component;
import java.util.Arrays;

@Aspect
@Component
@Order(1) // 对应图2：使用 @Order 控制多切面的执行顺序（数字越小优先级越高）
public class LoggingAspect {

    // 对应图3：使用 @Pointcut 定义可重用的切点表达式
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {} // 方法体为空，仅作为标记

    // 获取方法签名信息（对应图4）
    private String getMethodSignature(JoinPoint joinPoint) {
        return joinPoint.getSignature().toShortString();
    }
    
    // 获取方法参数信息（对应图4）
    private String getMethodArgs(JoinPoint joinPoint) {
        return Arrays.toString(joinPoint.getArgs());
    }
}
```

### 二、 五种通知类型代码示例

下面我们在 `LoggingAspect`类中添加五种通知的具体实现。

#### 1. 前置通知 - `@Before`

在目标方法**执行之前**执行，适用于参数校验、权限检查等。

```
@Before("serviceLayer()") // 引用切点
public void logBefore(JoinPoint joinPoint) {
    // 对应图4：通过 JoinPoint 获取连接点信息
    System.out.println("[前置通知] 准备执行方法: " + getMethodSignature(joinPoint));
    System.out.println("[前置通知] 方法参数: " + getMethodArgs(joinPoint));
}
```

**执行时机**：

```
[前置通知] 准备执行方法: UserService.login(..)
[前置通知] 方法参数: [admin, 123456]
--- 目标方法执行 ---
```

#### 2. 后置通知 - `@AfterReturning`

在目标方法**成功执行完毕后**执行（未发生异常），适用于记录操作结果。

```
@AfterReturning(pointcut = "serviceLayer()", returning = "result")
public void logAfterReturning(JoinPoint joinPoint, Object result) {
    System.out.println("[后置通知] 方法执行成功: " + getMethodSignature(joinPoint));
    System.out.println("[后置通知] 返回值: " + result);
}
```

**执行时机**：

```
--- 目标方法执行 ---
[后置通知] 方法执行成功: UserService.login(..)
[后置通知] 返回值: true
```

#### 3. 环绕通知 - `@Around`

功能最强大的通知，可以**自定义**目标方法的执行时机，适用于性能监控、事务管理等。

```
@Around("serviceLayer()")
public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
    String methodName = pjp.getSignature().getName();
    
    // 目标方法执行前
    long startTime = System.currentTimeMillis();
    System.out.println("[环绕通知] 开始执行方法: " + methodName);
    
    try {
        // 执行目标方法（可控制是否执行）
        Object result = pjp.proceed();
        
        // 目标方法成功执行后
        long endTime = System.currentTimeMillis();
        System.out.println("[环绕通知] 方法 " + methodName + " 执行成功，耗时: " + (endTime - startTime) + "ms");
        
        return result;
    } catch (Exception e) {
        // 目标方法抛出异常后
        System.out.println("[环绕通知] 方法 " + methodName + " 执行异常: " + e.getMessage());
        throw e;
    }
}
```

**执行时机**：

```
[环绕通知] 开始执行方法: login
--- 目标方法执行 ---
[环绕通知] 方法 login 执行成功，耗时: 15ms
```

#### 4. 异常通知 - `@AfterThrowing`

在目标方法**抛出异常后**执行，适用于异常处理、报警等。

```
@AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
public void logAfterThrowing(JoinPoint joinPoint, Exception ex) {
    System.out.println("[异常通知] 方法执行异常: " + getMethodSignature(joinPoint));
    System.out.println("[异常通知] 异常信息: " + ex.getMessage());
}
```

**执行时机**：

```
--- 目标方法执行（抛出异常） ---
[异常通知] 方法执行异常: UserService.login(..)
[异常通知] 异常信息: 用户名或密码错误
```

#### 5. 最终通知 - `@After`

在目标方法**执行结束后**执行，**无论是否发生异常**都会执行，适用于资源清理。

```
@After("serviceLayer()")
public void logAfter(JoinPoint joinPoint) {
    System.out.println("[最终通知] 方法执行结束: " + getMethodSignature(joinPoint));
}
```

**执行时机**：

```
--- 目标方法执行（无论成功或异常） ---
[最终通知] 方法执行结束: UserService.login(..)
```

### 三、 完整示例演示

假设我们有一个简单的业务类：

```
@Service
public class UserService {
    public boolean login(String username, String password) {
        System.out.println("用户登录验证: " + username);
        // 模拟业务逻辑
        if ("admin".equals(username) && "123456".equals(password)) {
            return true;
        }
        throw new RuntimeException("用户名或密码错误");
    }
}
```

**测试代码**：

```
@SpringBootTest
class UserServiceTest {
    @Autowired
    private UserService userService;
    
    @Test
    void testLoginSuccess() {
        userService.login("admin", "123456");
    }
    
    @Test
    void testLoginFailure() {
        try {
            userService.login("admin", "wrong");
        } catch (Exception e) {
            // 捕获异常
        }
    }
}
```

**正常执行输出**：

```
[环绕通知] 开始执行方法: login
[前置通知] 准备执行方法: UserService.login(..)
用户登录验证: admin
[后置通知] 方法执行成功: UserService.login(..)
[环绕通知] 方法 login 执行成功，耗时: 15ms
[最终通知] 方法执行结束: UserService.login(..)
```

**异常执行输出**：

```
[环绕通知] 开始执行方法: login
[前置通知] 准备执行方法: UserService.login(..)
用户登录验证: admin
[异常通知] 方法执行异常: UserService.login(..)
[最终通知] 方法执行结束: UserService.login(..)
[环绕通知] 方法 login 执行异常: 用户名或密码错误
```

