

主要回答三个问题“**我是什么类型的应用？**”、“**有哪些扩展组件要我处理？**”、“**我的起点在哪里？**”。
- **看依赖**（检查Classpath）→ 知道自己是什么类型的应用（**SERVLET/REACTIVE/NONE**）。
- **看配置**（读取`spring.factories`等）→ 知道自己有哪些扩展组件要处理（**Initializers, Listeners**）。
- **看栈帧**（分析调用栈）→ 知道自己的起点在哪里（**主配置类**）


---

1.  收集信息：获取主配置类。
	- 【主配置类】告诉 Spring Boot **“从哪里开始”**
		-  扫描组件（`@ComponentScan`）
		- 加载配置
2.  推断应用类型：
	- 推断方式：检查类路径下是否存在特定的类，自动推断你正在创建的是哪种类型的应用
	- 判定条件
		- SERVLET（传统的Spring MVC应用）：存在 `javax.servlet.Servlet`和 `org.springframework.web.context.ConfigurableWebApplicationContext`等类
		- REACTIVE（如Spring WebFlux应用）：存在 `org.springframework.web.reactive.DispatcherHandler`等类
		- NONE（非 Web 应用，如后台作业）：以上都不存在
	- 作用：
		- 决定了Spring Boot后续将为你的应用创建何种类型的`ApplicationContext`
		- 直接影响到**内嵌Web服务器**的选择
3.  加载扩展：Spring Boot SPI（服务提供者接口）机制，这是“自动配置”和“开箱即用”能力的核心
	- 核心拓展点：
		- 引导注册初始化器：`BootstrapRegistryInitializer`接口的实现类
			- 作用：向一个非常初级的 `BootstrapRegistry`中注册一些需要在极早期就可用的单例（如配置源解析器）
			- 调用时机：在 `ApplicationContext`和 `Environment`被创建之前
		- 初始化器:`ApplicationContextInitializer`接口的实现类
			- 作用：在容器正式启动前，对上下文进行一些自定义的配置
			- 调用时机：在 `ApplicationContext`被创建之后、但被刷新（refresh）之前被调用
		- 监听器:`ApplicationListener`接口的实现类
			- 作用：预留接口。框架和开发者可以通过监听这些事件，在特定的生命周期节点插入自定义逻辑。
			- 监听范围： Spring Boot 在整个启动过程中发布的各种事件
				- ApplicationStartingEvent`：最早的事件，在 `Environment`和 `Context`创建前触发。
			    - `ApplicationEnvironmentPreparedEvent`：`Environment`已准备就绪，但 `Context`尚未创建。
			    - `ApplicationPreparedEvent`：`Context`已加载（Bean定义已加载），但尚未刷新（Bean未实例化）。
			    - `ApplicationReadyEvent`：应用已完全启动，可以正常处理请求。
			    - `ApplicationFailedEvent`：启动过程中出现异常。
	- 文件读取的位置：
		- springboot2：`META-INF/spring.factories` 
		- springboo3:META-INF/spring/ 下的imports文件，按照扩展点类型分类
			 - 自动配置类对应：AutoConfiguration.imports
			- ApplicationContextInitializer对应ApplicationContextInitializer.imports
			- FailureAnalyzer对应：FailureAnalyzer.imports
	- 加载的方式：调用 `getSpringFactoriesInstances` 方法完成：获取实现类名 -> 反射创建实例 -> 排序
	
4. 推断主应用类（deduceMainApplicationClass()）
	- 推断方式：通过分析当前调用栈，找出包含 `main`方法的那个类。
	- **作用**：
		- **组件扫描的基准包**：`@SpringBootApplication`注解包含了`@ComponentScan`，其默认的扫描范围就是主配置类所在的包及其子包，这决定了哪些你标注了`@Component`, `@Service`等注解的Bean会被自动发现并注册
		- **日志和内部标识**：在日志中清晰地显示应用的启动类，并在某些内部操作中作为参考
 5. 完成封装：将所有初始化结果封装在一个配置完备的 `SpringApplication` 实例中。





![[Pasted image 20251117095827.png]]