 ![[Pasted image 20251116085951.png]]

----



## 一、 总览：Spring Boot 启动的核心两步

图1和图2高屋建瓴地指出了 Spring Boot 启动过程的两个核心动作：

1.  实例化 `SpringApplication` 对象：
    ```java
    // 对应图2中的第一行代码
    SpringApplication(primarySources) // 构造函数调用
    ```
这一步是准备阶段，负责初始化各种配置和环境。
- [[实例化SpringApplication完成的准备]]

2.  执行 `run` 方法：
    ```java
    // 对应图2中的第二行代码
    new SpringApplication(primarySources).run(args) // 启动运行
    ```
    这一步是执行阶段，真正地启动容器、加载Bean、启动内嵌服务器等。

您的图片重点深入剖析了第一个动作——`SpringApplication` 的实例化过程。

---

## 二、 深入 `SpringApplication` 实例化过程

1.  收集信息：获取主配置类。
2.  推断环境：自动判断应用类型（Servlet/Reactive/None）。
3.  加载扩展：通过 SPI 机制从 `META-INF/spring.factories` 中加载所有可用的初始化器和监听器，为后续的容器刷新和自动配置做好准备。
4.  完成封装：将所有初始化结果封装在一个配置完备的 `SpringApplication` 实例中。

### 1. 构造函数调用链

`SpringApplication` 的实例化通过一个构造函数链完成，其核心是资源加载器与主配置类的设置。

[[无界通配符#^ebb293]]

```java
// 图3: 公开的构造函数，接收主配置类（如 DemoApplication.class）
public SpringApplication(Class<?>... primarySources) {
    this(null, primarySources); // 调用核心私有构造函数
}

// 图4: 核心私有构造函数，完成实际初始化工作
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    this.resourceLoader = resourceLoader; // [1] 设置资源加载器
    // ... 其他关键操作
}
```
![[Pasted image 20251116090816.png]]
### 2. 核心初始化步骤详解

构造函数是初始化过程的灵魂，它按顺序完成了以下几件至关重要的事情：

#### a. 设置主配置源（Primary Sources）
```java
this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
```
- **设置主配置源**：将传入的主配置类（`primarySources`）存储在一个 `Set`集合中。这个集合非常重要，它告诉了 Spring Boot **“从哪里开始”** 扫描组件（`@ComponentScan`）和加载配置。

#### b. 推断应用类型（WebApplicationType.deduceFromClasspath()）
这是Spring Boot“约定大于配置”的典范。它通过检查类路径下是否存在特定的类，自动推断你正在创建的是哪种类型的应用：
•   SERVLET（如传统的Spring MVC应用）

•   REACTIVE（如Spring WebFlux应用）

•   NONE（非Web应用，即纯后台作业或命令行应用）
![[Pasted image 20251116091657.png]]
- 如果存在 `javax.servlet.Servlet`和 `org.springframework.web.context.ConfigurableWebApplicationContext`等类，则判定为 **SERVLET**（传统的 Spring MVC 应用）。
    
- 如果存在 `org.springframework.web.reactive.DispatcherHandler`等类，且不存在 Servlet 相关类，则判定为 **REACTIVE**（Spring WebFlux 响应式应用）。
    
- 如果以上都不存在，则判定为 **NONE**（非 Web 应用，如后台作业）。
![[Pasted image 20251116091846.png]]
#### c.getBootstrapRegistryInitializerFromSpringFactory

 ![[Pasted image 20251116092340.png]]
#### d. 加载 `spring.factories` 中的扩展组件（最关键的一步）
SpringBoot3不是这样的了，而是这样的了：
新机制把配置文件都统一放到 META-INF/spring/ 目录下，最贴心的是**按扩展点类型拆分**，一种扩展点对应一个文件，再也不用挤在一个文件里乱糟糟的了。比如：

- 自动配置类对应：AutoConfiguration.imports
    
- ApplicationContextInitializer对应：ApplicationContextInitializer.imports
    
- FailureAnalyzer对应：FailureAnalyzer.imports

![[Pasted image 20251117095827.png]]

这是Spring Boot SPI（服务提供者接口）机制的核心，也是“自动配置”的基石。
```java
// 获取初始化器
setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
// 获取监听器
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
```
•   作用：Spring Boot 会扫描所有 Jar 包中 `META-INF/spring.factories` 文件，读取其中注册的 `ApplicationContextInitializer` 和 `ApplicationListener` 接口的实现类。
![[Pasted image 20251116092404.png]]
•   流程：通过 `getSpringFactoriesInstances` 方法完成：获取实现类名 -> 反射创建实例 -> 排序。
![[Pasted image 20251116092145.png]]
- 意义：这是 **Spring Boot SPI（服务提供者接口）机制**的实现，是自动配置的基石。这使得第三方 Starter 或框架可以通过简单的配置文件，轻松地将自己的初始化逻辑或事件监听器嵌入到 Spring Boot 的启动生命周期中，实现了极高的可扩展性。
	- 这些**初始化器**会在 `ApplicationContext`**被创建之后、但被刷新（refresh）之前**被调用。用于在容器正式启动前，对上下文进行一些自定义的配置。
	- 这些**监听器**用于监听 Spring Boot 在整个启动过程中发布的**各种事件**（如 `ApplicationStartingEvent`, `ApplicationPreparedEvent`, `ApplicationReadyEvent`等）。框架和开发者可以通过监听这些事件，在特定的生命周期节点插入自定义逻辑。

#### e. 推断主应用类（deduceMainApplicationClass()）
```java
this.mainApplicationClass = deduceMainApplicationClass(); // 推断主应用类
```
- **操作**：通过分析当前调用栈，找出包含 `main`方法的那个类。
    
- **作用**：主要是为了日志记录和某些内部操作，能够清晰地知道应用是从哪个类启动的。
---

准备阶段是一个**信息搜集和预案准备**的过程。它通过 **SPI 机制**（`spring.factories`）**最大限度地发现了所有潜在的、可用的扩展点**（初始化器和监听器），并根据类路径**智能推断出了应用的运行环境**（应用类型）。这一切准备，都是为了在接下来的 `run()`方法中，能够有条不紊地、按需地创建和配置真正的 Spring 容器。

简单来说，这个阶段回答了三个问题：“**我是什么类型的应用？**”、“**有哪些扩展组件要我处理？**”、“**我的起点在哪里？**”。

----


## 三、深入理解run（）初始化容器

1.计时和判断是否答应日志
![[Pasted image 20251116102017.png]]![[Pasted image 20251116102032.png]]
![[Pasted image 20251116102050.png]]![[Pasted image 20251116102449.png]]
headless:

监听器listeners相关的操作本质上是预留的接口，让你拓展用的。


![[Pasted image 20251116103327.png]]



---



图1的代码是 Spring Boot `SpringApplication`类中 `run(String... args)`方法的**核心骨架**，它揭示了容器初始化的完整链条。

### 1. 启动计时与引导上下文创建

```
StopWatch stopWatch = new StopWatch();
stopWatch.start(); // 开始计时，用于监控启动性能
DefaultBootstrapContext bootstrapContext = createBootstrapContext();
```

- `StopWatch`：用于精确测量启动耗时，并在日志中输出。
    
- `createBootstrapContext()`：创建**引导上下文**。这是一个在真正的 `ApplicationContext`创建之前使用的、轻量级的上下文，主要用于 **Spring Cloud**等框架在极早期进行一些配置的引导和注册。
    

### 2. 配置 Headless 模式

解释了 `configureHeadlessProperty()`方法的作用。

- **是什么？**：Headless 模式是 Java 的一种运行模式，它允许在**没有物理显示设备、键盘或鼠标**的环境下（如服务器、命令行程序）运行需要图形操作的代码。
    
- **为什么需要？**：一些第三方库（如图表生成库、旧版的 AWT/Swing 组件）可能会尝试访问图形设备。在服务器环境下，这会导致 `java.awt.HeadlessException`异常。
    
- **Spring Boot 的做法**：图2中的代码确保系统属性 `java.awt.headless`被设置为 `true`。
    
    ```
    System.setProperty("java.awt.headless", 
        System.getProperty("java.awt.headless", "true")); // 默认值设为true
    ```
    
- **结论**：**Headless 模式的作用是“模拟”一个虚拟的图形环境，避免因服务器缺少真实图形设备而导致的启动失败**，确保应用在无界面环境中稳定运行。
    

### 3. 准备环境

图4的 `prepareEnvironment()`方法是**配置加载的核心**。

```
private ConfigurableEnvironment prepareEnvironment(...) {
    // 1. 创建环境对象（如StandardServletEnvironment）
    ConfigurableEnvironment environment = getOrCreateEnvironment(); 
    // 2. 加载配置：从命令行、application.properties/yml等所有配置源加载属性
    configureEnvironment(environment, applicationArguments.getSourceArgs()); 
    // 3. 将配置属性绑定到Spring环境
    ConfigurationPropertySources.attach(environment);
    // 4. 【关键】通知所有监听器：环境已准备就绪！
    listeners.environmentPrepared(bootstrapContext, environment);
    // ... 其他后处理
    return environment;
}
```

这一步完成后，所有在 `application.yml`等文件中配置的属性都已加载到 Spring 的 `Environment`中，可以被后续步骤使用。

### 4. 创建并刷新上下文- 核心中的核心

这是 Spring 容器初始化的**灵魂**。

- `createApplicationContext()`：根据之前推断的应用类型（SERVLET/REACTIVE/NONE），**实例化对应的 `ApplicationContext`**（如 `AnnotationConfigServletWebServerApplicationContext`）。
    
- `prepareContext()`：为刚创建的应用上下文进行**“装修”**。包括：
    
    - 设置环境（`Environment`）。
        
    - 执行 `BeanDefinition`的加载（扫描 `@Component`、处理 `@Bean`方法等）。
        
    - 触发 `ApplicationContextInitializer`。
        
    - 通知监听器上下文已加载。
        
    
- `refreshContext(context)`：**这是最复杂、最关键的一步**，它调用 `AbstractApplicationContext.refresh()`方法。这个过程包括：
    
    1. **准备 Bean 工厂**。
        
    2. **调用 BeanFactory 后置处理器**（如：解析 `@Configuration`类的 `ConfigurationClassPostProcessor`）。
	    **此阶段触发[[SpringBoot的自动装配原理|自动装配]]**。`ConfigurationClassPostProcessor`解析配置类，处理 `@EnableAutoConfiguration`注解。
        
    3. **注册 Bean 后置处理器**（如：处理 `@Autowired`的 `AutowiredAnnotationBeanPostProcessor`）。
        
    4. **初始化消息源、事件广播器**。
        
    5. **实例化所有非懒加载的单例 Bean**（依赖注入在此发生）。
        
    6. **完成刷新，发布 `ContextRefreshedEvent`事件**。
        
    7. **对于 Web 应用，在此步骤内嵌的 Web 服务器（如 Tomcat）会被启动**。
        
    

### 5. 收尾工作

- `afterRefresh()`：一个空的钩子方法，留给子类扩展。
    
- `callRunners(context, applicationArguments)`：执行所有 `ApplicationRunner`和 `CommandLineRunner`接口的实现类，用于应用启动后立即执行的逻辑。
    

---

### 二、 监听器（Listeners）的作用与本质**

图1中的 `listeners`（`SpringApplicationRunListeners`）和图6的“监听器类型”共同揭示了监听器的奥秘。

**1. 监听器是什么？**

监听器是实现了 `SpringApplicationRunListener`接口的组件。它们的主要职责是**在 Spring Boot 启动生命周期的各个关键节点发布相应的事件**。

**2. “本质是预留的接口，让你拓展用的” 这句话如何理解？**

这意味着 Spring Boot 的启动过程不是一个封闭的黑盒，而是通过**事件驱动架构**，向开发者提供了大量的**“钩子”（Hook Points）**。

- **工作原理**：如图1所示，在启动的每个重要阶段（如 `starting`, `environmentPrepared`, `contextPrepared`, `started`, `running`），都会调用 `listeners`的对应方法，从而发布一个事件。
    
- **如何拓展？**：作为开发者，你可以**创建自定义的监听器**，来监听这些事件，并在特定阶段插入你的业务逻辑。
    

**举例说明（对应图6的事件）：**

- **场景**：你需要在数据库连接池创建后、但应用正式接收请求前，预加载一些热点数据到缓存。
    
- **实现**：你可以创建一个监听器，监听 `ApplicationReadyEvent`事件（对应图6的第5条）。
    
    ```
    @Component
    public class MyCacheWarmUpListener implements ApplicationListener<ApplicationReadyEvent> {
        @Override
        public void onApplicationEvent(ApplicationReadyEvent event) {
            // 应用已完全就绪，此时执行缓存预热逻辑
            // 例如，从数据库加载热点数据并放入Redis
        }
    }
    ```
    

**3. 监听器类型（图6详解）**

图6清晰地列出了最重要的六种事件及其触发时机，它们构成了完整的启动生命周期通知：

1. `ApplicationStartingEvent`：**最早的事件**，在环境准备之前触发。
    
2. `ApplicationEnvironmentPreparedEvent`：**环境已准备好**，可以读取配置了。
    
3. `ApplicationPreparedEvent`：**Bean 定义已加载**，但 Bean 实例还未创建。
    
4. `ApplicationStartedEvent`：**上下文已刷新**，Bean 已实例化，内嵌服务器已启动。
    
5. `ApplicationReadyEvent`：**所有 Runner 已执行**，应用正式可处理请求。**（最常用）**
    
6. `ApplicationFailedEvent`：**启动过程中发生异常**，用于错误处理和通知。
    

### 三、 补充：配置忽略 BeanInfo（图5）


    

### 总结

这六张图连起来，完整揭示了 Spring Boot 容器初始化的魔法：

1. **流程清晰**：从计时开始，经过环境准备、上下文创建与刷新，到最后通知就绪，是一个**精心设计的流水线**。
    
2. **细节完善**：考虑了**无头模式**以保障服务器环境兼容性，并通过**忽略 BeanInfo**等操作进行性能优化。
    
3. **高度可扩展**：其核心在于**事件驱动架构**。通过**监听器机制**，在整个启动链路上预留了丰富的扩展点，使开发者和框架可以轻松地介入启动过程，实现自定义逻辑，这正是 Spring Boot 设计哲学中“开放扩展”的完美体现。
    

理解这个流程和其中的设计思想，是掌握 Spring Boot 原理和进行高级定制的基础。

