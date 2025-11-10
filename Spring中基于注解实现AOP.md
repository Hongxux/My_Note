---
aliases:
  - Spring + AspectJ（基于注解）
---
### 一、准备工作
要使用 Spring 的 AOP 功能，需要完成两个层面的配置：项目依赖和 Spring 容器配置。
#### 一、 添加 Maven 依赖（pom.xml）

图片上半部分列出了必须引入的库，这些库为 AOP 提供了运行时的支持。

```xml
<!-- 1. spring-context: Spring核心容器 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.0-M2</version>
</dependency>

<!-- 2. spring-aop: Spring AOP 核心框架 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aop</artifactId>
    <version>6.0.0-M2</version>
</dependency>

<!-- 3. spring-aspects: 集成AspectJ的支撑库 -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-aspects</artifactId>
    <version>6.0.0-M2</version>
</dependency>
```

依赖关系解读：
•   `spring-context`：是基础，它包含了 Spring 核心容器，是所有 Spring 应用的基础。

•   `spring-aop`：提供了 Spring 自身的 AOP 框架，包括代理机制等。

•   `spring-aspects`：至关重要。它包含了 AspectJ 的依赖，让 Spring 能够识别和处理诸如 `@Aspect`、`@Before`、`@After` 等注解，是实现注解驱动 AOP 的关键。


#### 二、 配置 Spring 配置文件（XML）

图片下半部分展示了如何在 Spring 的 XML 配置文件中声明必要的命名空间。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/aop
        http://www.springframework.org/schema/aop/spring-aop.xsd">
    <!-- 配置内容将在这里编写 -->
</beans>
```

命名空间解读：
•   `xmlns:context="..."`：此命名空间允许我们使用 `<context:component-scan>` 等标签，用于自动扫描和注册带有 `@Component`、`@Service` 等注解的 Bean。这是使用注解方式的基础。

•   `xmlns:aop="..."`：核心命名空间。它的声明意味着我们可以在配置文件中使用 AOP 相关的标签，最重要的是：

```xml
<!-- 启用基于注解的AOP功能 -->
<aop:aspectj-autoproxy/>
```
这行配置会告诉 Spring 容器：自动为被 `@Aspect` 注解标记的类创建代理对象，从而实现 AOP 功能。

结果：完成以上配置后，你就可以在代码中使用 `@Aspect` 来定义切面，使用 `@Before`、`@After`、`@Around` 等注解来定义通知（Advice），Spring 容器会自动处理代理的创建和织入。

现代 Spring Boot 项目的简化：
在当今主流的 Spring Boot 项目中，这些配置被极大简化：
•   依赖：通常只需在 `pom.xml` 中引入 `spring-boot-starter-aop`，它会自动管理所有必要的依赖。

•   配置：无需编写 XML 配置，Spring Boot 已自动配置好了 AOP 支持。你只需要直接编写 `@Aspect` 注解的类即可。

### 二、构建切面（切点＋通知）

这是需要被增强的原始业务类。

```
@Service("userService") // 声明这是一个Service层的Bean
public class UserService {
    public void login(){ 
        // 核心业务逻辑
        System.out.println("系统正在进行身份认证...."); 
    }
}
```

- 它本身只关心核心业务（身份认证），对日志记录等交叉逻辑一无所知。这就是 AOP 的**解耦**优势。

这是整个流程的触发点。 

```
public class SpringAOPTest {
    @Test
    public void testBefore(){
        // 1. 加载配置文件，初始化Spring容器
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring.xml");

        // 2. 从容器中获取UserService Bean
        // 注意：这里获取到的已经不是原始的UserService对象了，而是一个被Spring增强过的代理对象！
        UserService userService = applicationContext.getBean("userService", UserService.class);

        // 3. 调用方法
        userService.login();
    }
}
```
#### 1. 切面定义 - `LogAspect`
这是 AOP 的核心，它定义了**在何时（通知类型）、何地（切点）、执行什么增强逻辑（通知）**。
整体框架是：定义一个切面类（用@Aspect  标注这个类），如何在这个类中写切点（`@[通知的类型]("execution([切点表达式])")`）+通知（一个方法）

示例：
```
@Component("logAspect") // 将该类声明为Spring容器管理的Bean
@Aspect                 // 声明这是一个切面类
public class LogAspect {

    // 使用@Before注解定义一个前置通知
    // 参数是切点表达式，指定了增强逻辑的执行位置
    @Before("execution(* com.powernode.spring6.service.UserService.*(..))")
    public void 增强(){ 
        // 这是增强逻辑/通知
        System.out.println("我是一个通知，我是一段增强代码....");
    }
}
```
- [[AOP五种通知类型]]
- [[多切面执行顺序]]@order
- 在通知中获得当前正在执行的目标方法的信息[[joinpoint]]
- [[切点的重用]]
#### 2. Spring 配置 - `spring.xml`
这个配置文件开启了对 AOP 的支持。

```
<!-- 1. 开启组件扫描，让Spring能发现带注解的Bean -->
<context:component-scan base-package="com.powernode.spring6.service"/>

<!-- 2. 开启AspectJ注解自动代理，这是关键！ -->
<aop:aspectj-autoproxy proxy-target-class="true"/>
```

- **`<context:component-scan>`**：让 Spring 自动扫描 `com.powernode.spring6.service`包下的 `@Component`, `@Service`, `@Aspect`等注解，并创建相应的 Bean。
    
- **`<aop:aspectj-autoproxy>`**：**核心配置**。它指示 Spring 容器自动为带有 `@Aspect`注解的 Bean 创建代理对象。`proxy-target-class="true"`表示强制使用 CGLIB 库生成代理（即使目标类实现了接口）。
    



