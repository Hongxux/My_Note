这些注解用于标记一个类，让 Spring 在扫描包时能自动识别并将其创建为 Bean 实例放入容器中。它们的功能在本质上相似，但通过名称来区分不同的软件层次，使代码的职责更加清晰

。

|注解|适用场景|说明|
|---|---|---|
|**`@Component`**|通用组件|最基础的注解，可用于任何层次的类。当某个类不属于其他特定层次时，使用此注解<br><br>。|
|**`@Service`**|业务逻辑层（Service）|用于标记业务逻辑层的实现类，表明该类包含业务规则和逻辑<br><br>。|
|**`@Repository`**|数据访问层（DAO）|用于标记数据访问层（持久层）的组件。它的一个额外优点是能够将平台特定的异常（如 JDBC 抛出的 `SQLException`）转换为 Spring 统一的异常体系<br><br>。|
|**`@Controller`**|Web 控制层（MVC）|用于标记 Spring MVC 框架中的控制器，主要负责处理 Web 请求<br><br>。|
|**`@RestController`**|RESTful Web Service|是 `@Controller`和 `@ResponseBody`注解的组合，专门用于构建 RESTful 风格的 Web 服务，其内部的方法返回值默认直接写入 HTTP 响应体<br><br>。|

> **重要提示**：要使以上注解生效，必须在 Spring 的配置中启用组件扫描。例如，在 XML 配置中使用 `<context:component-scan base-package="你的包名"/>`，或在 Java 配置类上使用 `@ComponentScan`注解
> 
> 。




在项目中的实际应用（对应图中架构图）

这体现了经典的**三层架构**，每层使用对应的注解：

```
// 1. 数据访问层（Dao/Mapper）: 使用 @Repository
@Repository
public class UserDaoImpl implements UserDao {
    public User findById(Long id) { ... }
}

// 2. 业务逻辑层（Service）: 使用 @Service
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private UserDao userDao; // 依赖Dao
    // ... 业务方法
}

// 3. 控制层（Controller）: 使用 @Controller 或 @RestController
@RestController // @RestController = @Controller + @ResponseBody
public class UserController {
    @Autowired
    private UserService userService; // 依赖Service

    @GetMapping("/user/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```