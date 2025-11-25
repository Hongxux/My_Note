### 一、 应用类型与环境推断

这是所有准备的基石，决定了后续所有组件的行为模式。

1. **应用类型（WebApplicationType）**：
    
    - 通过 `WebApplicationType.deduceFromClasspath()`方法推断。
        
    - 结果有三种：`SERVLET`（传统 Web MVC）、`REACTIVE`（WebFlux）、`NONE`（非 Web 应用）。
        
    - **影响**：这个类型直接决定了后续将创建何种 `ApplicationContext`（如 `AnnotationConfigServletWebServerApplicationContext`）以及是否初始化 Web 服务器。
        
    
2. **主配置源（Primary Sources）**：
    
    - 存储了传递给 `SpringApplication`构造函数的入口类（如你的 `Application.class`）。
        
    - **作用**：这是后续组件扫描（`@ComponentScan`）的**起点**，告诉 Spring 从哪里开始寻找 Bean。
        
    

---

### 二、 扩展组件的加载（SPI 机制的运用）

这是“自动配置”和“开箱即用”能力的核心。Spring Boot 通过读取 `META-INF/spring.factories`文件，加载三大类扩展组件。

#### 1. **BootstrapRegistryInitializer**（引导注册初始化器）

- **加载时机**：最早被加载。
    
- **职责**：在 `ApplicationContext`和 `Environment`被创建**之前**，向一个非常初级的 `BootstrapRegistry`中注册一些**需要在极早期就可用的单例**（如配置源解析器）。
    
- **典型应用**：Spring Cloud 等高级框架，用于在容器生命周期的超早期进行配置。
    

#### 2. **ApplicationContextInitializer**（应用上下文初始化器）

- **加载时机**：在 `ApplicationContext`已被实例化**之后**，但在其 `refresh()`方法被调用**之前**。
    
- **职责**：用于在容器正式刷新前，**对 ConfigurableApplicationContext 进行编程式的自定义配置**。你可以通过它设置一些属性、激活 Profile，或者做一些自定义的准备工作。
    
- **典型应用**：
    
    java
    
    下载
    
    复制
    
    运行
    
    ```
    // 一个自定义的初始化器示例
    public class MyInitializer implements ApplicationContextInitializer<ConfigurableApplicationContext> {
        @Override
        public void initialize(ConfigurableApplicationContext context) {
            // 在容器刷新前，设置一个自定义的环境属性
            context.getEnvironment().getSystemProperties().put("custom.property", "value");
        }
    }
    ```
    
    - 需要在 `META-INF/spring.factories`中声明：`org.springframework.context.ApplicationContextInitializer=com.example.MyInitializer`
        
    

#### 3. **ApplicationListener**（应用事件监听器）

- **加载时机**：在准备阶段被加载并注册到即将创建的 `ApplicationContext`中。
    
- **职责**：监听 Spring Boot 在整个启动生命周期中发布的**各种事件**，并在特定时间点执行回调逻辑。这是实现**事件驱动架构**和**启动生命周期扩展**的关键。
    
- **重要事件**：
    
    - `ApplicationStartingEvent`：最早的事件，在 `Environment`和 `Context`创建前触发。
        
    - `ApplicationEnvironmentPreparedEvent`：`Environment`已准备就绪，但 `Context`尚未创建。
        
    - `ApplicationPreparedEvent`：`Context`已加载（Bean定义已加载），但尚未刷新（Bean未实例化）。
        
    - `ApplicationReadyEvent`：应用已完全启动，可以正常处理请求。
        
    - `ApplicationFailedEvent`：启动过程中出现异常。
        
    
- **典型应用**：执行启动时的数据初始化、监控启动过程、在特定阶段记录日志等。
    

---

### 三、 其他关键配置的推断

1. **主应用类（Main Application Class）**：
    
    - 通过 `deduceMainApplicationClass()`方法从调用栈中推断出包含 `main`方法的类。
        
    - **作用**：主要用于日志打印和一些内部逻辑，明确启动源头。
        
    
2. **命令行参数（CommandLineArgs）**：
    
    - 虽然解析主要在 `run()`方法中，但准备阶段会为处理 `args`参数做好准备。