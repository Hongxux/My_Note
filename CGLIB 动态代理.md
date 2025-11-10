 

### 一、 CGLIB 是什么？

> **CGLIB 是一个强大的、高性能的代码生成库，它可以在运行时动态生成类的子类，从而实现对类的代理。**

它与 JDK 动态代理的核心区别在于：

|特性|**JDK 动态代理**|**CGLIB 动态代理**|
|---|---|---|
|**实现机制**|基于**接口**|基于**继承**|
|**被代理目标**|必须实现至少一个接口|**普通类**或接口均可|
|**性能**|调用速度较快|生成代理类速度通常更快|

### 二、 核心特性与限制

1. **继承机制**：CGLIB 通过**生成目标类的子类**来创建代理。因此，代理对象就是目标对象的一个“子类实例”。
    
2. **目标类限制**：正因为基于继承，**被代理的类不能被 `final`修饰**（否则无法生成子类），同时，目标类中的方法也不能是 `final`的（否则无法重写）。
    

### 三、 开发步骤详解

#### 1. 引入依赖

首先，需要在项目中引入 CGLIB 库。

```
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>3.3.0</version>
</dependency>
```

#### 2. 准备目标类

定义一个没有实现接口的普通类 `UserService`，它就是我们要代理的目标。

```
public class UserService {
    public void login() {
        System.out.println("用户正在登录系统....");
    }
    public void logout() {
        System.out.println("用户正在退出系统....");
    }
}
```

#### 3. 实现方法拦截器- **核心**

这是 CGLIB 的“大脑”，负责插入增强逻辑。需要实现 `MethodInterceptor`接口。

```
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;
import java.lang.reflect.Method;

public class TimerMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        // 1. 前置增强：方法执行前
        long begin = System.currentTimeMillis();

        // 2. 调用目标类的原始方法
        // 注意：这里使用 proxy.invokeSuper(obj, args) 而不是 method.invoke()
        Object retVal = proxy.invokeSuper(obj, args);

        // 3. 后置增强：方法执行后
        long end = System.currentTimeMillis();
        System.out.println(method.getName() + "方法耗时 " + (end - begin) + " 毫秒");

        // 4. 返回原始方法的返回值
        return retVal;
    }
}
```

**关键点**：

- `proxy.invokeSuper(obj, args)`：这是**调用父类（即目标类）原始方法**的正确方式，比使用反射 `method.invoke()`效率更高。
    

#### 4. 创建代理对象- **组装**

使用 CGLIB 的 `Enhancer`类来创建代理对象。

```
import net.sf.cglib.proxy.Enhancer;

public class Client {
    public static void main(String[] args) {
        // 1. 创建 Enhancer 对象（相当于JDK代理中的Proxy类）
        Enhancer enhancer = new Enhancer();

        // 2. 设置父类（即要代理的目标类）
        enhancer.setSuperclass(UserService.class);

        // 3. 设置回调（即我们的方法拦截器）
        enhancer.setCallback(new TimerMethodInterceptor());

        // 4. 创建代理对象
        // 此时，CGLIB会在内存中动态生成UserService的一个子类
        // 并实例化这个子类对象作为代理返回
        UserService userServiceProxy = (UserService) enhancer.create();

        // 打印代理对象的类名（通常包含 $$EnhancerByCGLIB$$ 标志）
        System.out.println("代理对象类名: " + userServiceProxy.getClass().getName());

        // 5. 使用代理对象调用方法，增强逻辑会自动执行
        userServiceProxy.login();
        userServiceProxy.logout();
    }
}
```

### 四、 可能遇到的问题与解决

在高版本 JDK（如 17+）中运行 CGLIB，可能会因为**模块系统**的强封装性而抛出 `IllegalAccessError`。解决方案是在 JVM 启动参数中添加 `--add-opens`选项，开放相应模块的深度反射权限。

**在 IDEA 中的配置位置**（图5）：

`Run/Debug Configurations`-> 对应的应用 -> `VM options`：

```
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/sun.net.util=ALL-UNNAMED
```

![[Pasted image 20251107153249.png]]