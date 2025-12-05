1. 开启计时与引导上下文创建
	- 开启计时方式：使用StopWatch
		- 用于精确测量启动耗时，并在日志中输出。
	- 创建**引导上下文**：调用`createBootstrapContext()`
		- 这是一个在真正的 `ApplicationContext`创建之前使用的、轻量级的上下文，主要用于 **Spring Cloud**等框架在极早期进行一些配置的引导和注册。
2. 配置[[Headless]]模式：调用`configureHeadlessProperty()`方法
3. 准备环境对象：加载各种配置并触发 `ApplicationEnvironmentPreparedEvent`事件
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
	-  【配置包括】 `application.yml`、命令行参数、系统环境变量等，
	- 合并方式：按照优先级合并（例如，命令行参数可以覆盖配置文件中的设置）。
4. 创建应用上下文：调用`createApplicationContext()`方法
	- 【应用上下文的类型】之前推断的应用类型（SERVLET/REACTIVE/NONE）
	- 实例化对应的 `ApplicationContext`（如 `AnnotationConfigServletWebServerApplicationContext`）。
5.  准备应用上下文：调用`prepareContext()`
	- 设置环境（`Environment`）。第三步创建好的
	- 将主配置类（你的 `@SpringBootApplication`注解类）注册到容器中，执行 `BeanDefinition`的加载（扫描 `@Component`、处理 `@Bean`方法等）。
	- 触发 `ApplicationContextInitializer`，执行所有 `ApplicationContextInitializer`的初始化方法。
	- 通知监听器上下文已加载。
6. 刷新上下文：调用 `AbstractApplicationContext.refresh()`方法，控制了从配置准备 → BeanFactory 构建 → Bean 创建 → 事件发布的全流程

    1. 获取 Bean 工厂：`obtainFreshBeanFactory()`
	    1. 执行`this.createBeanFactory()`默认bean工厂：DefaultListableBeanFactory实例创建
	    2. 加载BeanDefinitions
		    - 这里的Bean来自于xml配置的
			    - 根据xml属性加载BeanDefinition
			    - 核心处理组件：XmlBeanDefinitionReader
		    - 注解配置的Bean在invokeBeanFactoryPostProcessors()​ 方法中发现和注册
			    - 核心处理组件：ConfigurationClassPostProcessor
		3. 返回对应的DefaultListableBeanFactory工厂实例
	2. BeanFactory 预准备:`prepareBeanFactory()`
	3. 准备和调用BeanFactory 后置处理器（BeanFactoryPostProcessors）
		- 设计目的：允许开发者在Bean定义加载完成后、但在Bean实例化之前，对Bean的元数据（BeanDefinition）进行修改
			- 例如，`ConfigurationClassPostProcessor`就是在这里解析 `@Configuration`, `@Bean`, `@Import`等注解，驱动了Spring Boot的自动配置
		1. 添加相关 Bean 后处理器,为后续创建好的 bean 可做扩展:`postProcessBeanFactory()`
	    2. 调用 BeanFactory 后置处理器:`invokeBeanFactoryPostProcessors()`
		    1.  执行 `BeanDefinitionRegistryPostProcessor`（可修改 Bean 定义）
		    2.  执行 `BeanFactoryPostProcessor`（可修改 BeanFactory 配置）
    4. 注册 Bean 后置处理器： `registerBeanPostProcessors()`
	    - 此步骤会找到所有实现了 `BeanPostProcessor`接口的Bean定义，并将它们注册到容器中
	    - 作用: 在Bean实例化之后、初始化前后进行拦截
		    - AOP、@Autowired、@ConfigurationProperties 等功能都依赖它
    5. 初始化消息源、事件广播器。
	    1. ` initMessageSource()`：国际化
			-  如果用户定义了 `messageSource` Bean，则使用用户的
			- 否则创建 `DelegatingMessageSource` 作为默认实现
		2.  `initApplicationEventMulticaster()`：事件广播器
			- 如果用户定义了 `applicationEventMulticaster`，则使用用户的
			-  否则创建默认的 `SimpleApplicationEventMulticaster`
	6.  子类刷新逻辑：`onRefresh()`
		- Web 环境下启动 `Tomcat/Jetty/Undertow` 内嵌容器
		-  加载 DispatcherServlet 等 MVC 核心组件
	7. 注册监听器： `registerListeners()`
		-  注册所有 `ApplicationListener` 类型的 Bean
		- 处理启动前缓存的早期事件
    8. 实例化所有非懒加载的单例 [[Bean的生命周期|Bean]]：`finishBeanFactoryInitialization()`
		- 是IoC和DI发生的核心阶段
	    1. 根据已有的 **BeanDefinition** 注册对应的bean
	    2. 依赖注入
	    3. 初始化方法执行（`@PostConstruct`、`InitializingBean` 等）
    9. 完成刷新：finishRefresh()`
	    1. 清理缓存
	    2. 发布 `ContextRefreshedEvent`
	    3. 标记启动完成
7. 收尾工作
	- `afterRefresh()`：一个空的钩子方法，留给子类扩展。
	- `callRunners(context, applicationArguments)`：执行所有 `ApplicationRunner`和 `CommandLineRunner`接口的实现类，用于应用启动后立即执行的逻辑。
		- 这里非常适合执行一些应用启动后需要立刻进行的初始化任务，比如预热缓存、初始化数据等
	    