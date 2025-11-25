
|属性名|主要作用|常用值与示例|
|---|---|---|
|**`webEnvironment`**|**定义测试的Web环境**，是控制测试行为的关键属性。|- **`MOCK`**(默认值)：模拟Servlet环境，不启动真实服务器。  <br>- **`RANDOM_PORT`**：启动真实嵌入式服务器并在随机端口上监听。  <br>- **`DEFINED_PORT`**：启动真实服务器并使用配置文件中定义的端口（如`server.port`）。  <br>- **`NONE`**：不提供任何Web环境，适用于纯业务层测试。|
|**`classes`**|**显式指定要加载的配置类**。当测试上下文无法自动找到主配置类时，或需要自定义测试配置时使用。|`@SpringBootTest(classes = {MyCustomConfig.class})`。若不指定，默认会自动搜索并加载主应用类（带`@SpringBootApplication`的类）。|
|**`properties`/ `value`**|**为测试环境添加或覆盖配置属性**。这两个属性是互为别名的。|`@SpringBootTest(properties = {"spring.datasource.url=jdbc:h2:mem:testdb", "my.prop=value"})`。|
|**`args`**|**模拟传递给应用的命令行参数**。|`@SpringBootTest(args = "--app.mode=test")`。|

### 💡 实用技巧与注意事项

1. **如何选择 Web 环境模式？**
    
    - 如果你想快速测试 Controller 层的逻辑，但不希望启动整个服务器，可以使用 **`MOCK`** 模式并结合 `MockMvc`。
        
    - 如果你需要进行完整的端到端集成测试（例如，使用 `TestRestTemplate`发起真实HTTP请求），则应选择 **`RANDOM_PORT`** 或 **`DEFINED_PORT`**。
        
    
2. **`classes`属性的典型场景**
    
    这个属性在你需要创建一个轻量级的、与主应用配置不同的测试上下文时特别有用。例如，你的主应用可能配置了数据库连接，但某个测试只想验证一个不依赖数据库的简单服务，这时就可以通过 `classes`属性只加载该服务所需的配置。
    
3. **提升测试速度**
    
    - **上下文缓存**：Spring Boot 会缓存测试上下文，相同配置的测试类会共享同一个上下文，避免了重复启动，从而提升测试速度。
        
    - **避免过度使用**：对于单一组件或方法的简单逻辑测试，直接使用JUnit进行单元测试，或者使用更细化的注解（如`@WebMvcTest`, `@DataJpaTest`）会更快，因为它们只加载应用程序的一部分，而不是像`@SpringBootTest`那样加载整个应用上下文。
        
    
4. **JUnit版本差异**
    
    在 Spring Boot 2.2.0 及更高版本中，由于默认集成的是 JUnit 5（Jupiter），因此可以直接使用 `@SpringBootTest`，无需再配合 `@RunWith`注解。如果你使用的是较早版本或需要兼容 JUnit 4，则可能需要加上 `@RunWith(SpringRunner.class)`。
    

### 💎 总结

简单来说，`@SpringBootTest`的核心属性让你能精细控制测试的“世界”是如何构建的：`webEnvironment`决定是否启动Web服务器及如何启动，`classes`决定加载哪些配置，而 `properties`和 `args`则用于微调运行时的配置和行为。希望这些解释能帮助你更好地运用这个强大的测试工具。