## 声明式Bean加载（业务开发）
### 第一部分：声明式Bean 加载的演变


#### 1. 原始方式：纯 XML 配置

这是最早期 Spring 框架的方式，所有 Bean 都在一个庞大的 XML 文件中声明。

```
<bean id="bookService" class="com.itheima.service.impl.BookServiceImpl" scope="singleton"/>
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"/>
```

- **特点**：**集中式配置**，所有依赖关系在 XML 中明确写出。
    
- **优点**：配置与代码分离，理论上的控制力最强。
    
- **缺点**：**极其繁琐**，XML 文件庞大，且不是类型安全的（拼写错误在运行时才发现）。
    
- **现状**：在现代 Spring Boot 开发中**已基本被淘汰**，仅用于集成非常古老的系统或特殊需求。
    

#### 2. 过渡方案：XML 扫描 + 注解声明

随着注解的普及，Spring 2.5 引入了组件扫描，进入“半注解”时代。

- **图2 - XML 中开启扫描**：
    
    ```
    <context:component-scan base-package="com.itheima"/>
    ```
    
    这行配置告诉 Spring：请扫描 `com.itheima`包及其子包下所有带有 `@Component`等注解的类，并将它们自动注册为 Bean。
    
- **图3 - 使用注解声明 Bean**：
    
    - **声明自定义 Bean**：使用**原型注解**。
        
        ```
        @Service // 等价于@Component，但语义更清晰（服务层）
        public class BookServiceImpl implements BookService {}
        ```
        
        `@Component`、`@Service`、`@Controller`、`@Repository`功能相同，用于标记不同层次的组件，便于阅读和维护。
        
    - **声明第三方 Bean**：在配置类中使用 `@Bean`注解。
        
        ```
        @Component
        public class DbConfig {
            @Bean // 方法返回的对象将被注册为Bean
            public DruidDataSource dataSource() {
                return new DruidDataSource();
            }
        }
        ```
        
    
- **特点**：**配置与代码混合**。大大减少了 XML 的配置量，是向全注解驱动开发的重要过渡。
    
- **现状**：现在仍在使用，但 XML 部分正逐渐被完全替代。
    

#### 3. 现代方式：全注解配置

这是当前 **Spring Boot 推荐和默认使用的方式**，完全告别 XML。

```
@Configuration // 1. 标记此为配置类（替代XML文件）
@ComponentScan("com.itheima") // 2. (可选) 开启组件扫描，Boot默认已扫描主类所在包
public class SpringConfig {
    @Bean // 3. 注册第三方Bean
    public DruidDataSource getDataSource() {
        return new DruidDataSource();
    }
}
```

- **特点**：**类型安全、简洁、强大**。利用 Java 代码的优势，配置信息可被 IDE 重构、检查，极大提升了开发效率和可靠性。
    
- **`@Configuration`说明**：如果配置类本身不会被其他配置扫描（例如在 Spring Boot 中直接通过 `@Import`导入），则该注解可省略。但通常保留以明确其职责。
#### 4.低耦合的Bean注册方式：`@Import`注解直接加载Bean

`@Import`注解的主要作用是：**在配置类上使用，直接告诉Spring容器：“请将指定的一个或多个Class注册为Bean”。**
	而Bean对象上无需声明注解声明为Bean

1. **在配置类上添加`@Import`注解**：
    
    ```
    @Configuration
    @Import(Dog.class)  // 关键：在此处导入要注册的Bean类
    public class SpringConfig {
        // ... 其他配置
    }
    ```
    
2. **被导入的类无需任何注解**：
    
    ```
    // 注意：这个类本身没有任何Spring注解！
    public class Dog {
        // ... 类的普通定义
    }
    ```
- **`Dog`类是一个“纯净”的Java类**，它没有任何Spring相关的注解。这意味着你可以在非Spring项目中使用它，测试它，而无需引入任何Spring依赖。
    
- 决定将其纳入Spring容器管理的“权力”被转移到了**配置类**中。这使得应用程序的**核心领域模型（如`Dog`类）与Spring框架解耦**。

正因为这种低耦合的特性，`@Import`注解在 **“spring技术底层及诸多框架的整合中大量使用”**。

**典型场景举例**：

1. **整合第三方库**：当你要将一个第三方库中的类（你无法修改其源码，自然无法给它加 `@Component`注解）配置为Spring Bean时，`@Import`是完美选择。
    
2. **Spring Boot自动配置**：Spring Boot 的自动配置正是大量使用了 `@Import`注解（通常通过 `@EnableAutoConfiguration`触发），来条件性地加载一系列配置类及其所需的Bean。
    
3. **模块化配置**：通过 `@Import`可以将多个配置类组织在一起，实现模块化的配置管理。
    

---

### 第二部分：Bean 加载的高阶扩展方案


#### 1. 扩展1：`FactoryBean`- 复杂的 Bean 初始化

当 Bean 的创建过程非常复杂（如需要大量初始化、依赖其他服务）时，直接写在 `@Bean`方法里会显得臃肿。`FactoryBean`接口提供了解决方案。

```
// 1. 实现FactoryBean接口，泛型为要生产的Bean类型
public class BookFactoryBean implements FactoryBean<Book> {
    @Override
    public Book getObject() throws Exception {
        Book book = new Book();
        // ... 复杂的初始化逻辑，如读取文件、网络请求等
        return book;
    }
    @Override
    public Class<?> getObjectType() {
        return Book.class; // 返回要创建的Bean类型
    }
}

// 2. 注册FactoryBean本身
@Configuration
public class SpringConfig {
    @Bean
    public BookFactoryBean book() { // 注意：这里注册的是FactoryBean
        return new BookFactoryBean();
    }
}
// 3. 实际效果：Spring容器中会存在两个Bean
//    - 名为"book"的Bean，类型是BookFactoryBean（工厂本身）
//    - 名为"&book"的Bean（在名字前加&），类型是Book（工厂生产的产品）
```

- **应用场景**：整合 MyBatis 时，`SqlSessionFactoryBean`就是最经典的 `FactoryBean`，它负责创建复杂的 `SqlSessionFactory`。
    

#### 2. 扩展2：`@ImportResource`- 系统迁移与兼容

用于**将旧的、基于 XML 配置的 Spring 项目平滑迁移到 Spring Boot**注解配置的项目中，实现新旧配置的共存。

```
@Configuration
@ComponentScan("com.itheima")
@ImportResource("classpath:applicationContext-config.xml") // 引入旧的XML配置文件
public class SpringConfig2 {
    // 新的注解配置可以在这里添加
}
```

- **应用场景**：**项目重构或迁移期**。允许你逐步将 XML 配置替换为 Java 配置，而不是一次性重写。
    

#### 3. 扩展3：`proxyBeanMethods`- 优化 Bean 的调用

这是 Spring Boot 中的一个重要优化特性，与 `@Configuration(proxyBeanMethods = true/false)`配合使用。

- **问题**：在 `@Configuration`类中，一个 `@Bean`方法调用另一个 `@Bean`方法时，你期望的是从容器中获取单例，还是每次创建一个新实例？
    
- **解决方案**：`proxyBeanMethods`参数。
    
    - **`proxyBeanMethods = true`（默认值，Full 模式）**：
        
        ```
        @Configuration(proxyBeanMethods = true) // 默认就是true，可省略
        public class SpringConfig3 {
            @Bean
            public Book book() {
                return new Book();
            }
            @Bean
            public Library library() {
                // 此处调用book()方法，会被Spring拦截，
                // 返回的是容器中的单例Bean，而不是每次new一个新的Book。
                return new Library(book());
            }
        }
        ```
        
        **优点**：保证 Bean 的单例性。
        
        **缺点**：运行时需要通过 CGLIB 代理，对启动性能有微小影响。
        
    - **`proxyBeanMethods = false`（Lite 模式）**：
        
        ```
        @Configuration(proxyBeanMethods = false) // 设置为Lite模式
        public class SpringConfig3 {
            // ... 同上
        }
        ```
        
        **优点**：启动更快，无需代理。
        
        **缺点**：`@Bean`方法相互调用时，每次都会执行方法体（如同普通 Java 方法），**无法保证单例**。适用于**无依赖的、独立的 Bean**配置。
        
    

## 编程式Bean加载（框架开发、拓展）
与声明式（如 `@Component`, `@Bean`）不同，编程式加载让你通过**编写 Java 代码**来动态地、有条件地向容器中注册 Bean，提供了极高的灵活性和控制力。

|特性|**`ImportSelector`**|**`ImportBeanDefinitionRegistrar`**|**`BeanDefinitionRegistryPostProcessor`**|
|---|---|---|---|
|**核心职责**|**动态选择**要导入的配置类|**编程式注册**Bean定义|**最终干预**Bean定义注册表|
|**控制粒度**|中粒度（类级别）|**细粒度**（Bean定义级别）|**超细粒度**（容器级别）|
|**执行时机**|配置类解析阶段|配置类解析阶段|**容器刷新阶段**（所有配置解析后）|
|**输出**|返回**类的全限定名字符串数组**|直接操作`BeanDefinitionRegistry`|直接操作`BeanDefinitionRegistry`|
|**能力**|根据条件返回不同的配置类名|可**注册、修改、移除**Bean定义|可**注册、修改、移除、覆盖**任何Bean定义|
|**典型应用**|条件化配置（如`@Conditional`）|框架整合（如MyBatis Mapper注册）|**Spring Boot自动配置**、全局覆盖|

### 三种编程式Bean加载

---

#### 一、 `ImportSelector`：条件选择器

##### 1. 核心机制

- **作用时机**：同样在处理 `@Import`注解时，如果引用的是 `ImportSelector`实现类，会调用其 `selectImports`方法。
    
- **核心能力**：该方法返回一个**类的全限定名数组**。Spring 容器会**自动将这些类注册为 Bean**。这使得导入行为可以根据条件动态变化。

##### 2. 代码解读（对应图2）


```java
public class MyImportSelector implements ImportSelector {
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        // 1. 获取导入源的元信息（如：判断配置类上是否有某些注解）
        boolean flag = importingClassMetadata.hasAnnotation("org.springframework.context.annotation.Import");

        // 2. 根据条件动态返回需要导入的类的全名
        if (flag) {
            return new String[]{"com.itheima.domain.Dog"}; // 导入Dog类
        } else {
            return new String[]{"com.itheima.domain.Cat"}; // 导入Cat类
        }
    }
}
```

**这段代码的效果是**：根据当前配置类是否被 `@Import`注解所标记，来决定向容器中注册 `Dog`还是 `Cat`这个 Bean。这是一种**条件化装配**。

##### 3. 特点与典型应用场景

- **特点**：**更简洁、更专注于“选择”逻辑**，无需关心 Bean 的具体注册细节。
    
- **应用场景**：
    
    - **环境切换**：最经典的例子是 Spring Boot 的 **`@Conditional`** 系列注解（如 `@ConditionalOnClass`, `@ConditionalOnProperty`）。它们的底层大量使用了 `ImportSelector`来根据类路径、配置属性等条件决定是否启用某些自动配置。
        
    - **模块化加载**：根据不同的特性开关，动态加载不同的功能模块。


    

---

#### 二、 `ImportBeanDefinitionRegistrar`：Bean定义注册器

**1. 核心思想：**

直接获取到 Spring 容器提供的 **`BeanDefinitionRegistry`**（Bean定义注册中心），从而可以编程式地**注册、修改或移除**任何一个 Bean 的定义。

**2. 实现方式：**

实现 `ImportBeanDefinitionRegistrar`接口，重写 `registerBeanDefinitions`方法。

```
public class MyBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 1. 构建一个Bean定义
        BeanDefinition beanDefinition = BeanDefinitionBuilder
                .rootBeanDefinition(MyService.class)
                .setScope(BeanDefinition.SCOPE_SINGLETON) // 可以设置作用域等属性
                .getBeanDefinition();
                
        // 2. 将其注册到容器中，并指定Bean名称
        registry.registerBeanDefinition("myService", beanDefinition);
        
        // 还可以移除或覆盖已有的Bean定义
        // registry.removeBeanDefinition("oldService");
    }
}
```

**3. 如何使用：**

同样通过 `@Import`注解引入。

```
@Configuration
@Import(MyBeanDefinitionRegistrar.class)
public class AppConfig {
}
```

**4. 特点与应用场景：**

- **细粒度控制**：直接操作 `BeanDefinition`，可以设置 Bean 的几乎所有元信息（如类、作用域、初始化方法等）。
    
- **框架整合的利器**：例如，**MyBatis**用它来扫描 Mapper 接口并动态注册为 Bean。它读取指定包路径，为每个接口生成一个代理类的 BeanDefinition 并注册。
    
- **典型场景**：需要将非 `@Component`注解的类（如第三方库的类、接口）动态注册为 Bean。
    

---

#### 三、 `BeanDefinitionRegistryPostProcessor`：容器后置处理器

**1. 核心思想：**

这是**功能最强大、优先级最高**的编程式加载方式。它在 Spring 容器**刷新**的早期被调用，此时**所有配置元数据（包括 `@Configuration`类、XML 等）都已被加载并解析**成 `BeanDefinition`，但还没有任何一个 Bean 被实例化。它允许你对最终的 BeanDefinition 集合进行**最后的干预和修改**。

**2. 实现方式：**

实现 `BeanDefinitionRegistryPostProcessor`接口，重写 `postProcessBeanDefinitionRegistry`方法。

```
@Component // 它本身需要是一个Bean，才能被容器识别并调用
public class MyBeanPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        // 此时可以查看和操作容器中所有的BeanDefinition
        if (registry.containsBeanDefinition("someService")) {
            // 获取已有的定义并进行修改...
        }
        
        // 或者注册新的BeanDefinition
        BeanDefinition definition = ...;
        registry.registerBeanDefinition("newService", definition);
    }
}
```

**3. 特点与应用场景：**

- **超细粒度控制**：可以操作容器中**任何一个**Bean 的定义，无论它来自何处（注解、XML等）。
    
- **终极武器**：**Spring Boot 自动配置**的核心机制。它检查类路径（如是否有某些 Jar 包）、环境变量等，然后动态注册所需的 Bean（如数据源、事务管理器等）。
    
- **典型场景**：进行全局性的 Bean 定义管理、覆盖其他配置源定义的 Bean、在框架层面进行深度定制。
    

### Bean加载控制（基于编程式加载Bean）
好的，这四张图片系统地阐述了 **Spring Boot 条件化装配（Conditional Bean Loading）** 的核心机制。它展示了如何使用一系列 `@ConditionalOn*`注解，根据特定条件来**动态地、智能地控制 Bean 的创建与否**，这是 Spring Boot “自动配置”和“约定大于配置”理念的基石。

下面我将为您整合这四张图的信息，提供一个从基础到高级的完整解析。

---

#### 一、 根据类路径是否存在某个类进行条件装配
**注解**：`@ConditionalOnClass`

```
public class SpringConfig {
    @Bean
    @ConditionalOnClass(Mouse.class) // 关键注解
    public Cat tom(){
        return new Cat();
    }
}
```

- **条件规则**：只有当**类路径下存在 `Mouse`这个类**时，Spring Boot 才会执行 `tom()`方法，将 `Cat`对象注册为一个 Bean。
    
- **工作原理**：Spring Boot 会检查当前的类加载器中是否能加载到指定的类。
    
- **典型应用场景**：这是 **Spring Boot Starter 自动配置的核心理念**。
    
    - **例如**：只有在项目中引入了 `spring-boot-starter-data-redis`依赖（该依赖包含了 `RedisTemplate`类），Spring Boot 的自动配置类中关于 `RedisTemplate`的 `@Bean`方法才会生效。这样就避免了因缺少依赖而报错。
        
    

---

#### 二、 根据容器中是否存在某个 Bean 进行条件装配

这两个图展示了 `@ConditionalOnBean`的两种用法。

##### 1. 根据 Bean 的类型匹配

**注解**：`@ConditionalOnBean`

```
@Import(Mouse.class) // 首先，通过@Import确保Mouse类被注册为Bean
public class SpringConfig {
    @Bean
    @ConditionalOnBean(Mouse.class) // 关键注解：按类型匹配
    public Cat tom(){
        return new Cat();
    }
}
```

- **条件规则**：只有当 Spring 容器中**已经存在一个 `Mouse`类型的 Bean**时，才会创建 `Cat`Bean。
    
- **注意**：图2中使用了 `@Import(Mouse.class)`来确保 `Mouse`先被注册，这样才能满足条件。在实际应用中，这个 `Mouse`Bean 可能来自其他配置类或自动配置。
    

##### 2. 根据 Bean 的名称匹配

**注解**：`@ConditionalOnBean(name = "...")`

```
@Import(Mouse.class)
public class SpringConfig {
    @Bean
    @ConditionalOnBean(name = "jerry") // 关键注解：按Bean的名称匹配
    public Cat tom(){
        return new Cat();
    }
}
```

- **条件规则**：只有当 Spring 容器中**已经存在一个名为 `"jerry"`的 Bean**时，才会创建 `Cat`Bean。
    
- **应用场景**：当你需要依赖一个特定名称的 Bean，而不是特定类型的 Bean 时。例如，系统中可能配置了多个同类型的数据源，你需要根据名称指定依赖其中一个。
    

---

#### 三、 复杂条件组合：根据环境特征进行装配

图4展示了最强大的用法：**将多个条件注解组合使用，实现精细化的环境控制**。

```
@Configuration
@Import(Mouse.class)
public class MyConfig {
    @Bean
    // 开始组合多个条件注解
    @ConditionalOnClass(Mouse.class) // 条件1：存在Mouse类
    @ConditionalOnMissingClass("com.itheima.bean.Dog") // 条件2：不存在Dog类
    @ConditionalOnNotWebApplication // 条件3：非Web应用
    public Cat tom() {
        return new Cat();
    }
}
```

**条件规则（组合逻辑为“与”关系）**：

只有当**所有**以下条件都满足时，`Cat`Bean 才会被创建：

1. **条件1**：类路径下存在 `Mouse`类。
    
2. **条件2**：类路径下**不存在**`com.itheima.bean.Dog`类。
    
3. **条件3**：当前应用**不是**一个 Web 应用（例如，不是一个 Spring MVC 或 Spring WebFlux 应用）。
    

- **逻辑关系**：多个 `@Conditional*`注解一起使用时，是 **“与”**的关系，即必须全部满足。
    
- **强大之处**：通过组合，可以精确地定义 Bean 的创建环境，这是 Spring Boot 为不同运行环境（如普通 Java 应用、Web 应用、测试环境）提供不同配置的基础。
    

---

#### 四、 总结：条件化装配的价值与常用注解

**1. 核心价值**

- **防止错误**：避免在缺少必要依赖时尝试创建 Bean，导致 `ClassNotFoundException`或 `NoSuchMethodError`。
    
- **自动配置**：是 Spring Boot “开箱即用” 的魔法来源。它根据你引入的 Jar 包（类路径），自动为你配置好相应的功能。
    
- **环境适配**：使同一份代码能轻松适应开发、测试、生产等不同环境。
    

**2. 常用条件注解家族**

除了图中的注解，Spring Boot 还提供了丰富的条件注解：

- `@ConditionalOnProperty`：根据配置文件（如 `application.yml`）中的属性值进行判断。
    
- `@ConditionalOnExpression`：通过 SpEL 表达式进行更复杂的判断。
    
- `@ConditionalOnJava`：根据 JVM 版本进行判断。
    
- `@ConditionalOnWebApplication`/ `@ConditionalOnNotWebApplication`：判断是否是 Web 应用。
    
- `@ConditionalOnMissingBean`：**与 `@ConditionalOnBean`相反**，只有当容器中**不存在**指定类型或名称的 Bean 时才生效。这在**提供默认配置**和**允许用户覆盖**的场景中极为常用。
    

**简单来说，这四张图由浅入深地揭示了 Spring Boot 的智能行为：它不是无脑地注册所有 Bean，而是像一个有经验的管家，会先检查“当前的环境是否具备条件”，再决定是否要“提供某项服务（Bean）”。** 掌握这些条件注解，是理解和定制 Spring Boot 自动配置的关键。