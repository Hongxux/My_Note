---
aliases:
  - Bean
---
**Spring Bean 的生命周期是一个精心设计的管道，在 Bean 被创建、使用和销毁的每一个关键节点，都预留了“钩子”（Hook），使得框架和开发者能够插入自定义逻辑，从而实现强大的功能。**[[Bean生命周期的扩展点]]
![[f82deb22c29d57202354d69a707487f1.jpg]]

#### 1. 解析XML配置文件和注解，封装Bean元数据到**BeanDefinition**对象
- 设计目的：Spring 容器需要知道要创建什么 Bean、如何创建、以及它们的依赖关系。这些信息被封装在 `BeanDefinition`对象中。
- **定义和扫描**：[[Bean的注册方式]]
- **封装信息与注册：** 将每个 Bean 的定义信息转换为 **BeanDefinition**对象（Bean 的“蓝图”）。
	这个对象封装了 Bean 的类名、作用域（单例/原型）、是否懒加载等元数据。然后将其进行注册。
####  2.实例化：通过反射调用构造方法或者指定工厂方法
- 特点：其属性均为默认值，依赖也还未注入（此时得到的只是一个“空壳”）
- 时机：受到Bean作用域影响
	- 单例 Bean 通常在容器启动时创建；
	- 而原型 Bean 则在每次被请求时创建新实例
- **扩展点**：`InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation()`方法会在此阶段**之前**执行。
	- 如果此方法返回一个非 null 对象，Spring 会**短路**后续默认的实例化及属性赋值流程，直接进入初始化阶段
#### 3. 属性赋值（依赖注入）
- 注入的对象：
	- `@Autowired`：Spring 框架提供的注解，是**最常用**的自动注入注解。它默认按**类型（ByType）**进行自动装配。
	- `@Resource`：属于 JSR-250 标准（Java 规范）。它默认按 **名称（ByName）**进行装配。如果按名称找不到，则会回退到按类型查找。
	- `@Inject`：属于 JSR-330 标准（Java 依赖注入标准），功能与 `@Autowired`类似。
- 可能存在的问题：循环依赖，
	- 解决的前提：循环依赖的双方中有一方是单例Bean，且不是在初始化函数中出现循环依赖
	- 解决措施：Spring 会通过**三级缓存**机制来解决。
- 扩展点：
	- InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()和 postProcessProperties()：方法会在此阶段前后被调用，允许对属性值进行修改
	- **Aware 接口回调**：发生在属性注入之后、初始化之前。**向 Bean“告知”一些容器内部信息**：
		- `BeanNameAware`：告知 Bean 它在容器中的名字（ID）。
		- `BeanClassLoaderAware`：告知加载该 Bean 的类加载器。
		- `BeanFactoryAware`：告知 Bean 工厂本身（一个更底层的容器对象）。		
#### 4.初始化：Bean 完全就绪前的最后准备阶段
1. **[[BeanPostProcessor]]的 前置处理**：所有 `BeanPostProcessor`的 `postProcessBeforeInitialization`方法被调用，允许在初始化前对所有 Bean 进行修改
    
2. **执行初始化方法**：按顺序执行三种初始化逻辑
    - `@PostConstruct`注解标记的方法。
    - `InitializingBean`接口的 `afterPropertiesSet()`方法。
    - 在 XML 或 `@Bean`注解中自定义的 `init-method`。
3. **BeanPostProcessor 后置处理**：所有 `BeanPostProcessor`的 `postProcessAfterInitialization`方法被调用。**AOP 代理对象通常就是在此阶段生成的**。
#### 5. **使用**：
- **Bean存放位置**： 已经完全初始化好，被存放于**单例池**中
- **获取方式**：应用程序可以通过容器获取并使用它
#### 6. **销毁**：
- **时机**：容器关闭时（如 `applicationContext.close()`）
- **清理逻辑**：执行一些自定义的清理逻辑，其顺序与初始化相反：
1. `@PreDestroy`注解标记的方法。
2. `DisposableBean`接口的 `destroy()`方法。
3. 自定义的 `destroy-method`。

