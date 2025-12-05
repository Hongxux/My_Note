主要回答三个问题“**我是什么类型的应用？**”、“**有哪些扩展组件要我处理？**”、“**我的起点在哪里？**”。

1.  收集信息：获取主配置类。
	- 【主配置类】告诉 Spring Boot **“从哪里开始”**
		-  扫描组件（`@ComponentScan`）
		- 加载配置
2.  推断环境：自动判断应用类型（Servlet/Reactive/None）。
	- 【推断方式】检查类路径下是否存在特定的类，自动推断你正在创建的是哪种类型的应用
	- 【应用类型】
		- SERVLET（传统的Spring MVC应用）：存在 `javax.servlet.Servlet`和 `org.springframework.web.context.ConfigurableWebApplicationContext`等类
		- REACTIVE（如Spring WebFlux应用）：存在 `org.springframework.web.reactive.DispatcherHandler`等类
		- NONE（非 Web 应用，如后台作业）：以上都不存在
	- 作用：决定了后续的run函数中创建哪一种context对象
3.  加载扩展：Spring Boot SPI（服务提供者接口）机制，这是“自动配置”和“开箱即用”能力的核心
	- 加载读取文件的位置：![[Pasted image 20251117095827.png]]
		- springboot2：`META-INF/spring.factories` 
		- springboo3:META-INF/spring/ 下的imports文件，按照扩展点类型分类
			 - 自动配置类对应：AutoConfiguration.imports
			- ApplicationContextInitializer对应ApplicationContextInitializer.imports
			- FailureAnalyzer对应：FailureAnalyzer.imports
	- 流程：调用 `getSpringFactoriesInstances` 方法完成：获取实现类名 -> 反射创建实例 -> 排序
		- 引导注册初始化器：`BootstrapRegistryInitializer`接口的实现类
			- 调用时机：在 `ApplicationContext`和 `Environment`被创建之前
			- 作用：向一个非常初级的 `BootstrapRegistry`中注册一些需要在极早期就可用的单例（如配置源解析器）
		- 初始化器:`ApplicationContextInitializer`接口的实现类
			- 调用时机：在 `ApplicationContext`被创建之后、但被刷新（refresh）之前被调用
			- 作用：在容器正式启动前，对上下文进行一些自定义的配置
		- 监听器:`ApplicationListener`接口的实现类
			- 监听范围： Spring Boot 在整个启动过程中发布的各种事件
				- ApplicationStartingEvent`：最早的事件，在 `Environment`和 `Context`创建前触发。
			    - `ApplicationEnvironmentPreparedEvent`：`Environment`已准备就绪，但 `Context`尚未创建。
			    - `ApplicationPreparedEvent`：`Context`已加载（Bean定义已加载），但尚未刷新（Bean未实例化）。
			    - `ApplicationReadyEvent`：应用已完全启动，可以正常处理请求。
			    - `ApplicationFailedEvent`：启动过程中出现异常。
			- 作用：预留接口。框架和开发者可以通过监听这些事件，在特定的生命周期节点插入自定义逻辑。
4. 推断主应用类（deduceMainApplicationClass()）
	- 推断方式：通过分析当前调用栈，找出包含 `main`方法的那个类。
	- **作用**：主要是为了日志记录和某些内部操作，能够清晰地知道应用是从哪个类启动的。
5.  完成封装：将所有初始化结果封装在一个配置完备的 `SpringApplication` 实例中。
