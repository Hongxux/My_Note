

### 一、 核心方法：`Proxy.newProxyInstance()`


- **作用**：用于**创建动态代理对象**。
    
- **底层执行了两步操作**：
    
    1. **在内存中动态生成代理类的字节码**：该代理类会实现指定的接口。
        
    2. **实例化代理对象**：根据生成的字节码，通过反射创建出代理对象的实例。
**简单说**：调用这个方法，JVM 会在运行时为你“凭空”创建一个新的类（代理类）并返回它的实例。
### 二、`Proxy.newProxyInstance()` 三个关键参数详解
Proxy.newProxyInstance()的三个参数

| 参数                          | 类型    | 作用与要求                                                                                                |
| --------------------------- | ----- | ---------------------------------------------------------------------------------------------------- |
| **`ClassLoader loader`**    | 类加载器  | 用于**加载**在内存中动态生成的代理类。**关键要求**：必须使用**目标类**的类加载器，即 `target.getClass().getClassLoader()`。               |
| **`Class[] interfaces`**    | 接口数组  | 指定动态生成的代理类需要**实现哪些接口**。**关键要求**：代理类和目标类必须实现相同的接口，这样才能实现“透明”替换，即 `target.getClass().getInterfaces()`。 |
| **[[InvocationHandler]]** | 调用处理器 | **这是核心！**是一个接口，你需要在它的实现类中编写**增强逻辑**（如日志、事务等）。JDK 不知道你要增强什么，所以把它设计成接口，由你来自定义实现。                       |

**代码示例**：

```
// target 是已有的目标对象
OrderService proxyObj = (OrderService) Proxy.newProxyInstance(
        target.getClass().getClassLoader(), // 参数1：类加载器
        target.getClass().getInterfaces(),   // 参数2：接口数组
        new TimerInvocationHandler(target)   // 参数3：调用处理器实例
);
```


    

### 三、 封装代理对象的获取

在实际开发中，我们通常会将创建代理对象的代码封装成一个工具类，这样更简洁、更易用。

**工具类封装**：

```
public class ProxyUtil {
    public static Object newProxyInstance(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new TimerInvocationHandler(target) // 传入目标对象
        );
    }
}
```

**使用工具类**：

```
// 原始目标对象
OrderService target = new OrderServiceImpl();
// 通过工具类轻松获取代理对象
OrderService proxy = (OrderService) ProxyUtil.newProxyInstance(target);
// 调用代理对象的方法，会自动触发增强逻辑
