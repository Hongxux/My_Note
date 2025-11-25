实在找不到就打印来看看


---

### 一、 使用 `@Component`及其衍生注解（最常用）

当你使用 `@Component`、`@Service`、`@Controller`、`@Repository`注解一个类时，Spring 会**自动扫描**并将其注册为 Bean。

**默认名称规则：将类名的首字母变为小写。**

|类名|默认 Bean 名称|代码示例|
|---|---|---|
|`BookService`|`bookService`|`@Service public class BookService { ... }`|
|`UserDAOImpl`|`userDAOImpl`|`@Repository public class UserDAOImpl { ... }`|
|`AController`|`aController`|`@Controller public class AController { ... }`|
|`MyCustomComponent`|`myCustomComponent`|`@Component public class MyCustomComponent { ... }`|

**验证方法：**

你可以在应用程序启动日志中看到这些信息，或者通过以下方式打印所有 Bean 的名称来验证：

```
@Autowired
private ApplicationContext applicationContext;

@PostConstruct
public void printBeanNames() {
    String[] beanNames = applicationContext.getBeanDefinitionNames();
    Arrays.sort(beanNames);
    for (String beanName : beanNames) {
        System.out.println(beanName);
    }
}
```

---

### 二、 使用 `@Bean`注解（在 `@Configuration`类中）

在配置类的方法上使用 `@Bean`注解时，Spring 会将方法的**返回值**注册为 Bean。

**默认名称规则：直接使用方法名作为 Bean 的名称。**

```
@Configuration
public class AppConfig {

    @Bean // 这个Bean的默认名称是 "dataSource"，与方法名一致
    public DataSource dataSource() {
        return new DruidDataSource();
    }

    @Bean // 这个Bean的默认名称是 "entityManagerFactory"
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        // ... 配置逻辑
        return factory;
    }
}
```

**重要提示：**这是最需要留意的地方。Bean 的名称**与方法返回的类型无关，只与方法名有关**。例如，一个返回 `String`类型的方法 `@Bean public String appName() { ... }`，其 Bean 名称是 `appName`，而不是 `string`。

---

### 三、 使用 XML 配置（传统方式）

在 XML 中使用 `<bean>`标签时，情况稍有不同。

**规则：如果没有指定 `id`或 `name`属性，Spring 会为该 Bean 生成一个唯一的、匿名的内部标识符**。这个标识符通常基于类的全限定名和计数器，例如 `com.example.MyBean#0`。

**不推荐**依赖这种匿名名称来获取 Bean，因为它不可读且不稳定。

**正确做法：总是显式指定 `id`或 `name`。**

```
<beans>
    <!-- 使用 id 属性（推荐，ID必须唯一） -->
    <bean id="bookService" class="com.example.BookServiceImpl"/>

    <!-- 使用 name 属性（可以指定多个别名，用逗号分隔） -->
    <bean name="bookServ, myBookService" class="com.example.BookServiceImpl"/>
</beans>
```

---

### 四、 如何自定义 Bean 的名称？

如果你不想使用默认名称，可以轻松地自定义。

#### 1. 为 `@Component`及其衍生注解自定义名称

直接在注解中提供值即可。

```
@Service("myBookService") // 自定义Bean名称为 "myBookService"
public class BookServiceImpl implements BookService {
    // ...
}
```

#### 2. 为 `@Bean`注解自定义名称

同样，在注解中提供值。

```
@Configuration
public class AppConfig {
    @Bean("mainDataSource") // 自定义Bean名称为 "mainDataSource"
    public DataSource dataSource() {
        return new DruidDataSource();
    }
}
```

---

### 总结与最佳实践

|声明方式|默认名称规则|自定义方式|
|---|---|---|
|**`@Component`, `@Service`等**|类名，**首字母小写**|`@Service("customName")`|
|**`@Bean`方法**|**方法名**|`@Bean("customName")`|
|**XML `<bean>`**|匿名内部标识符（不实用）|使用 `id`或 `name`属性|

**最佳实践建议：**

1. **保持默认**：对于简单的项目，使用默认名称通常就够了，而且更简洁。
    
2. **显式命名**：当需要更清晰的语义或有多个同类型 Bean 时，**推荐显式自定义名称**。例如，有两个数据源时，命名为 `@Bean("primaryDataSource")`和 `@Bean("secondaryDataSource")`会比默认名清晰得多。
    
3. **命名约定**：遵循 Spring 的惯例，使用**驼峰式命名法**（camelCase），如 `userService`, `orderRepository`。
    

理解 Bean 的默认命名规则，对于后续进行 Bean 的查找（如 `applicationContext.getBean("name")`）、按名称注入（`@Autowired @Qualifier("name")`）以及排除 Bean 冲突都至关重要。