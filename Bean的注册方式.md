
|加载方式|在生命周期中的定位与作用|
|---|---|
|**1. XML `<bean/>`**|最基础的元数据来源。容器启动时，**解析 XML 文件**，将每个 `<bean/>`标签转换为一个 `BeanDefinition`并注册。|
|**2. `@Component`及衍生注解**|容器在**组件扫描**过程中，发现带有这些注解的类，为每个类生成一个 `BeanDefinition`并注册。|
|**3. `@Bean`**|在解析 `@Configuration`类时，遇到 `@Bean`方法，会为其生成一个 `BeanDefinition`并注册。|
|**4. `@Import(MyClass.class)`**|在解析 `@Import`注解时，直接为指定的 `MyClass`生成一个 `BeanDefinition`并注册。|
|**5. `ImportSelector`**|在解析 `@Import`时，调用其 `selectImports`方法，**动态获取一批类的全限定名**，然后为每个类生成 `BeanDefinition`并注册。|
|**6. `ImportBeanDefinitionRegistrar`**|在解析 `@Import`时，调用其 `registerBeanDefinitions`方法，并传入 **`BeanDefinitionRegistry`**参数。开发者可以在此**直接、编程式地**注册、修改或移除任何 `BeanDefinition`。**这是对注册过程的精细干预。**|
|**7. `BeanDefinitionRegistryPostProcessor`**|这是**所有 `BeanDefinition`注册的【终极后门】**。在所有配置元数据（上述1-6）都被解析并注册后，容器会调用此接口的 `postProcessBeanDefinitionRegistry`方法。此时，开发者可以**检查并干预整个 `BeanDefinitionRegistry`**，可以覆盖任何已注册的 `BeanDefinition`，或注册新的。**Spring Boot 的自动配置就基于此实现。**|
