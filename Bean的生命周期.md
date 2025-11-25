---
aliases:
  - Bean
---
**Spring Bean 的生命周期是一个精心设计的管道，在 Bean 被创建、使用和销毁的每一个关键节点，都预留了“钩子”（Hook），使得框架和开发者能够插入自定义逻辑，从而实现强大的功能。**
![[f82deb22c29d57202354d69a707487f1.jpg]]
### Bean的创建
#### 1. 解析XML配置文件，封装Bean元数据到**BeanDefinition**对象
Spring 容器需要知道要创建什么 Bean、如何创建、以及它们的依赖关系。这些信息被封装在 `BeanDefinition`对象中。
- **定义和扫描**：Spring 容器启动时，会解析配置（如 XML、注解或 Java Config类）
	容器启动时，[[Bean的扫描|扫描]]指定包路径下的类，识别带有 `@Component`, `@Service`, `@Controller`, `@Repository`, `@Configuration`（会扫描其定义的@Bean）等注解的类，将其定义为 Bean。（[[定义Bean的注解]]）
- **封装信息与注册：** 将每个 Bean 的定义信息转换为 **BeanDefinition**对象（Bean 的“蓝图”）。
	这个对象封装了 Bean 的类名、作用域（单例/原型）、是否懒加载等元数据。然后将其进行注册。
	
|加载方式|在生命周期中的定位与作用|
|---|---|
|**1. XML `<bean/>`**|最基础的元数据来源。容器启动时，**解析 XML 文件**，将每个 `<bean/>`标签转换为一个 `BeanDefinition`并注册。|
|**2. `@Component`及衍生注解**|容器在**组件扫描**过程中，发现带有这些注解的类，为每个类生成一个 `BeanDefinition`并注册。|
|**3. `@Bean`**|在解析 `@Configuration`类时，遇到 `@Bean`方法，会为其生成一个 `BeanDefinition`并注册。|
|**4. `@Import(MyClass.class)`**|在解析 `@Import`注解时，直接为指定的 `MyClass`生成一个 `BeanDefinition`并注册。|
|**5. `ImportSelector`**|在解析 `@Import`时，调用其 `selectImports`方法，**动态获取一批类的全限定名**，然后为每个类生成 `BeanDefinition`并注册。|
|**6. `ImportBeanDefinitionRegistrar`**|在解析 `@Import`时，调用其 `registerBeanDefinitions`方法，并传入 **`BeanDefinitionRegistry`**参数。开发者可以在此**直接、编程式地**注册、修改或移除任何 `BeanDefinition`。**这是对注册过程的精细干预。**|
|**7. `BeanDefinitionRegistryPostProcessor`**|这是**所有 `BeanDefinition`注册的【终极后门】**。在所有配置元数据（上述1-6）都被解析并注册后，容器会调用此接口的 `postProcessBeanDefinitionRegistry`方法。此时，开发者可以**检查并干预整个 `BeanDefinitionRegistry`**，可以覆盖任何已注册的 `BeanDefinition`，或注册新的。**Spring Boot 的自动配置就基于此实现。**|
####  2.实例化：通过反射调用构造方法或者指定工厂方法

- 容器根据 `BeanDefinition`，通过**反射**调用 Bean 的构造方法或指定的工厂方法（BeanFactory）来创建对象实例。
	**其属性均为默认值，依赖也还未注入**（此时得到的只是一个“空壳”）
	

- **Bean作用域影响实例化的时机**：
	- 单例 Bean 通常在容器启动时创建；
	- 而原型 Bean 则在每次被请求时创建新实例
#### 3. 属性赋值（依赖注入）

- Spring 容器根据定义，为刚创建好的 Bean 实例**注入所需的属性和依赖**。
	- 这可以通过 `@Autowired`、`@Value`等注解，或 XML 中的 配置来实现
		- `@Autowired`：Spring 框架提供的注解，是**最常用**的自动注入注解。它默认按**类型（ByType）**进行自动装配。
		- `@Resource`：属于 JSR-250 标准（Java 规范）。它默认按 **名称（ByName）**进行装配。如果按名称找不到，则会回退到按类型查找。
		- `@Inject`：属于 JSR-330 标准（Java 依赖注入标准），功能与 `@Autowired`类似。
- 如果遇到循环依赖，Spring 会通过**三级缓存**机制来解决。
#### 4.初始化

这是生命周期中**最复杂且扩展点最多的阶段**，目的是让 Bean 达到就绪状态。其步骤有严格的顺序：

1. **Aware 接口回调**：
- 检查 Bean 是否实现了特定的 `Aware`接口，如果是，则调用接口方法，**向 Bean“告知”一些容器内部信息**：
	- `BeanNameAware`：告知 Bean 它在容器中的名字（ID）。
	- `BeanClassLoaderAware`：告知加载该 Bean 的类加载器。
	- `BeanFactoryAware`：告知 Bean 工厂本身（一个更底层的容器对象）。		
2. **[[BeanPostProcessor]] 前置处理**：所有 `BeanPostProcessor`的 `postProcessBeforeInitialization`方法被调用，允许在初始化前对 Bean 进行修改
    
3. **执行初始化方法**：按顺序执行三种初始化逻辑
    - `@PostConstruct`注解标记的方法（JSR-250 标准）。
    - `InitializingBean`接口的 `afterPropertiesSet()`方法。
    - 在 XML 或 `@Bean`注解中自定义的 `init-method`。
        
4. **BeanPostProcessor 后置处理**：所有 `BeanPostProcessor`的 `postProcessAfterInitialization`方法被调用。**AOP 代理对象通常就是在此阶段生成的**。
---


#### 5. **使用**：
- **Bean存放位置**： 已经完全初始化好，被存放于**单例池**中
- **获取方式**：应用程序可以通过容器获取并使用它
#### 6. **销毁**：
- **时机**：容器关闭时（如 `applicationContext.close()`）
- **清理逻辑**：执行一些自定义的清理逻辑，其顺序与初始化相反：
1. `@PreDestroy`注解标记的方法。
2. `DisposableBean`接口的 `destroy()`方法。
3. 自定义的 `destroy-method`。

#### **实例化方式决定生命周期管理的范围**

- **单例 Bean**：默认情况。实例化发生在**容器启动时**，后续完整的生命周期均由 Spring 容器管理，包括最终的销毁。
- **原型 Bean**：实例化发生在**每次被获取时**（（`getBean()`））。Spring 容器只负责实例化和依赖注入，**不管理其销毁阶段**
    

