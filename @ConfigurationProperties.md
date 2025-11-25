---
aliases:
  - "@ConfigurationProperties"
---
| 特性               | `@ConfigurationProperties`             | `@Value` |
| ---------------- | -------------------------------------- | -------- |
| **批量绑定**         | ✅ 支持                                   | ❌ 仅支持单个  |
| **复杂类型（如集合、嵌套）** | ✅ 支持良好                                 | ❌ 不支持    |
| **宽松绑定**         | ✅ 支持（如 `first-name`可绑定到 `firstName`字段） | ❌ 不支持    |
| **计量单位自动转换**     | ✅ 支持                                   | ❌ 不支持    |

### @ConfigurationProperties文件的编写规范
1. **定义配置类**:创建一个普通Java类，使用 `@ConfigurationProperties(prefix = "your.prefix")`注解标记。类的字段名及其类型与配置文件中 `prefix`之后的属性名和类型相对应
	- `@ConfigurationProperties`适用于**批量**绑定一组相关的配置，提供类型安全、松散绑定和验证等高级特性，是管理复杂配置的推荐方式
	- **使用 [[数据校验|@Validated]]进行验证**：在配置类上添加 `@Validated`注解，并在字段上使用校验注解（如 `@NotNull`, `@Min`, `@Max`），可以在应用启动时就捕获配置错误
	- **考虑添加 `spring-boot-configuration-processor`依赖**：在 `pom.xml`中添加此依赖（scope通常为 `optional`），可以在编写配置文件时获得IDE的自动补全和提示功能，极大提升开发体验
2. **使其生效**:让Spring管理这个配置类并注入属性值
3. **属性绑定**：应用启动时，Spring Boot会读取配置文件，根据指定的前缀和字段名，自动将配置值设置（注入）到配置类实例的对应字段中

这是一个标准的配置属性类，其作用是绑定配置文件（如 `application.yml`）中以 `servers`为前缀的属性。

```
//@Component // 此注解被注释掉
@Data
@ConfigurationProperties(prefix = "servers")
public class ServerConfig {
    // 该类会绑定诸如 servers.ipAddress, servers.port 等配置属性
}
```

- **`@ConfigurationProperties(prefix = "servers")`**： 核心注解，声明这个类用于接收配置。
    
- **`@Data`**： Lombok 注解，自动生成 getter、setter 等方法。
    
- **`@Component`**： 关键点！这个注解被注释了，意味着这个类**不会**通过组件扫描的方式被自动创建并加入到 Spring 容器中。

### @ConfigurationProperties 的两种启用方式


如何让一个配置属性类生效（**纳入Bean的管理**）。

####  两种启用方式的对比

为了让上面的 `ServerConfig`类生效（即被创建为 Bean 并能够注入使用），Spring Boot 提供了两种方式，**二者选其一即可**。

##### **方式一：在主启动类上使用 @EnableConfigurationProperties**

```
@SpringBootApplication
@EnableConfigurationProperties(ServerConfig.class) // 显式启用指定配置类
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

- **作用**：`@EnableConfigurationProperties`注解会显式地将指定的配置类（此处为 `ServerConfig.class`）注册为 Spring 容器中的一个 Bean。
    

##### **方式二：在配置类上直接使用 `@Component`**

如果将 `ServerConfig`类上的 `@Component`注解取消注释，那么它就会像普通的组件（如 `@Controller`, `@Service`）一样，被 Spring 的组件扫描自动发现并注册为 Bean。此时，主启动类上的 `@EnableConfigurationProperties(ServerConfig.class)`就可以去掉了。

### [[开启yml提示]]

