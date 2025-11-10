在通知方法中，我们通常需要获取当前正在执行的目标方法的信息（如方法名、参数等）。图4说明了如何通过 `JoinPoint`参数来实现。

- **参数类型**：在通知方法中声明一个 `JoinPoint`类型的参数。
    
- **自动注入**：Spring AOP 会自动将连接点信息注入到这个参数中。
    
- **常用方法**：
    
    - `joinPoint.getSignature()`：获取**方法签名**对象（`Signature`）。
        
    - `signature.getName()`：获取**目标方法名**。
        
    - `joinPoint.getArgs()`：获取**目标方法的参数列表**。
        
    

**示例**：

```
@Before("globalPointcut()")
public void beforeAdvice(JoinPoint joinPoint) { // 声明JoinPoint参数
    // 获取方法签名
    Signature signature = joinPoint.getSignature();
    // 获取方法名
    String methodName = signature.getName();
    // 获取参数
    Object[] args = joinPoint.getArgs();

    System.out.println("前置通知：正在调用方法 " + methodName + "，参数为：" + Arrays.toString(args));
}
```
