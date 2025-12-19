---
aliases:
  - IoC容器
---
- 需求背景：IoC的实现
- 解决措施：由容器负责对象的**创建、装配、生命周期管理**，开发者只需关注业务逻辑
- 实现方式：Spring采用**分层架构**实现IoC容器功能：
	- 配置元数据层：支持XML、注解和JavaConfig
	- 核心容器层：BeanFactory提供基础DI功能
		- 定位：Spring IoC容器的**最底层接口**，提供基础功能
		- 依赖注入时机：采用**延迟加载**策略
			- 只有在调用`getBean()`时才实例化Bean
		- 依赖注入方式：通过**Java反射**机制动态创建对象并注入依赖
		- 核心实现类：`DefaultListableBeanFactory`、`XmlBeanFactory`
	- 扩展容器层：ApplicationContext提供企业级特性
		- 定位：BeanFactory的**子接口**，是功能更丰富的"超集"
		- 依赖注入时机：**预加载**策略
			- 在容器启动时实例化所有单例Bean
			- 通过`refresh()`方法完成生命周期关键阶段，包括BeanFactory创建、后置处理及非延迟单例Bean实例化
		- 核心实现类：`ClassPathXmlApplicationContext`、`AnnotationConfigApplicationContext`
- BeanFactory和ApplicationContext的对比

| 特性维度   | BeanFactory            | ApplicationContext                                        |
| ------ | ---------------------- | --------------------------------------------------------- |
| 企业级功能  | 仅提供基础IoC和DI功能          | 提供国际化、事件发布/监听机制、便捷的统一资源访问、面向切面编程友好的AOP集成等企业级功能            |
| 后处理器注册 | 需手动注册BeanPostProcessor | **自动检测并注册**，对注解（如 `@Autowired`, `@Transactional`）的支持开箱即用。 |
| 错误检测   | 配置问题仅在运行时暴露            | 启动时即检验配置，提前发现配置问题和循环依赖                                    |
- **国际化支持**：ApplicationContext继承`MessageSource`接口，提供多语言消息访问能力
- **事件机制**：ApplicationContext内置事件发布订阅机制，支持组件间松耦合通信
- **资源访问**：ApplicationContext通过`ResourceLoader`接口统一访问各类资源
- **AOP集成**：ApplicationContext为面向切面编程提供更好支持，使`@Autowired`等注解"开箱即用"


- 问题：
	- **BeanFactory的局限性**
	    - **问题**：延迟加载导致配置问题难以提前发现
	- **ApplicationContext的资源消耗**
		- **问题**：
			- 预加载增加启动时间和内存消耗
		    - 使用反射（Reflection）机制来实例化对象和注入依赖，这比直接使用 `new`关键字创建对象会有一定的性能损失
		- **解决**：
			- 通过`lazy-init="true"`配置实现部分Bean的延迟初始化
			- 合理规划Bean作用域，减少不必要的单例Bean
			- 避免在构造函数中执行耗时操作
	- **循环依赖问题**
		- **问题**：Setter注入可能导致循环依赖
		- **解决**：Spring通过**三级缓存**策略解决：
