![[Bean生命周期有很多扩展点，企业开发哪些扩展点常常被使用，....docx]]
###  影响多个Bean的容器级扩展点

这类扩展接口的实现类是独立于 Bean 的，它们会注册到 Spring 容器中，对所有或指定的 Bean 创建过程进行干预 。

- **`BeanPostProcessor`（最重要、最常用）**：作用于 **初始化阶段前后**​ 。![[Pasted image 20251211143041.png]]
    
    - `postProcessBeforeInitialization`：在所有 Bean 初始化方法（如 `@PostConstruct`）**之前**​ 被调用。常用于修改 Bean 的属性或返回一个包装对象 。
        
    - `postProcessAfterInitialization`：在所有 Bean 初始化方法（如 `@PostConstruct`）**之后**​ 被调用。**Spring AOP 正是在此阶段创建代理对象**，这也是为什么增强逻辑能包裹住业务方法的原因 。
        
    - _常见实现_：`AutowiredAnnotationBeanPostProcessor`（处理 `@Autowired`, `@Value`注入）。
        
    
- **`InstantiationAwareBeanPostProcessor`**：它是 `BeanPostProcessor`的子接口，将干预点提前到了 **实例化阶段前后**​ 。
    
    - `postProcessBeforeInstantiation`：在 Bean **实例化之前**​ 调用。如果返回值不为 null，则会**跳过后续默认的实例化和属性赋值流程**，仅执行 `postProcessAfterInitialization`。可用于返回一个自定义的代理对象来代替目标 Bean 。
        
    - `postProcessAfterInstantiation`：在 Bean **实例化之后**，**属性赋值之前**​ 调用。返回值可控制是否进行后续的属性填充 。
        
    - `postProcessProperties`：在**属性赋值阶段**调用，允许你对即将设置的属性值（`PropertyValues`）进行修改 。
        
    
- **`SmartInstantiationAwareBeanPostProcessor`**：进一步扩展了实例化流程，用于处理更特殊的场景 。
    
    - `determineCandidateConstructors`：用于在多个构造器中选择合适的构造器。
        
    - `getEarlyBeanReference`：此方法在解决**循环依赖**时至关重要，用于从三级缓存返回一个早期引用 。
        
    
- **`MergedBeanDefinitionPostProcessor`**：用于在 Bean 实例化之后、初始化之前，对合并后的 Bean 定义（`RootBeanDefinition`）进行后处理，例如预解析 `@Autowired`等注解 。
    
- **`BeanFactoryPostProcessor`**：注意，它作用于 **Bean 定义级别**，在 Bean 实例化之前，允许我们修改应用程序上下文的 Bean 定义（`BeanDefinition`），比如修改 property 的值。它不属于 Bean 生命周期级别的扩展点，但功能非常强大 。
    

### 针对单个Bean的扩展点

这些接口需要由 Bean 类本身实现，因此只对实现了该接口的**当前这个 Bean**​ 生效 。

- **一系列 `Aware`接口**：这些接口用于让 Bean **感知**到 Spring 容器中的特定资源。它们的调用时机在 **属性赋值之后，初始化回调之前**​ 。
    
    - `BeanNameAware`：感知到其在容器中的名字。
        
    - `BeanFactoryAware`：感知到所属的 `BeanFactory`。
        
    - `ApplicationContextAware`：感知到所属的 `ApplicationContext`（可以获取容器中几乎所有资源）。
        
    - 其他如 `EnvironmentAware`, `ResourceLoaderAware`等。
        
    
- **初始化相关接口与注解**：
    - `@PostConstruct`注解：这是 JSR-250 标准注解，作用同 `afterPropertiesSet()`，是 Spring **官方推荐**的初始化方式 。
    - `InitializingBean`接口：实现该接口的 Bean 在属性设置完成后，会执行其 ![[Bean生命周期有很多扩展点，企业开发哪些扩展点常常被使用，....docx]]()`方法 。
    - 自定义 `init-method`：通过在 `@Bean`注解或 XML 中指定初始化方法。
    - **执行顺序**为：`@PostConstruct`-> `InitializingBean.afterPropertiesSet()`-> 自定义 `init-method`。
        
    
- **销毁相关接口与注解**：
    
    - `DisposableBean`接口：实现该接口的 Bean 在容器关闭时，会执行其 `destroy()`方法 。
        
    - `@PreDestroy`注解：JSR-250 标准注解，是 Spring **官方推荐**的销毁方式 。
        
    - 自定义 `destroy-method`：通过在 `@Bean`注解或 XML 中指定销毁方法。
        
    - **执行顺序**为：`@PreDestroy`-> `DisposableBean.destroy()`-> 自定义 `destroy-method`。
        
    

### 💡 实际应用场景

了解这些扩展点能帮助我们解决很多实际问题：

- **Spring AOP 的实现**：代理对象的创建主要在 `BeanPostProcessor`的 `postProcessAfterInitialization`阶段完成 。
    
- **注解的解析与注入**：如 `@Autowired`（由 `AutowiredAnnotationBeanPostProcessor`处理）和 `@PostConstruct`（由 `CommonAnnotationBeanPostProcessor`处理）都是通过相应的 `BeanPostProcessor`实现的 。
    
- **配置的加解密**：可以实现一个 `BeanPostProcessor`，在 `postProcessBeforeInitialization`阶段扫描 Bean 的属性，对加密的值进行解密。
    
- **自定义校验**：在 `InitializingBean.afterPropertiesSet()`方法中，检查依赖注入是否完整，配置是否正确。
    
- **资源管理**：在 `@PreDestroy`方法中，安全地关闭数据库连接、释放文件句柄等资源。
    
