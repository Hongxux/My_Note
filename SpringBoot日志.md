---
aliases:
  - 日志
---

### 1. 日志级别

日志的级别（从低到高/从详细到严重）：

- **TRACE**： 最详细的运行堆栈信息，使用率低。
    
- **DEBUG**： 主要供开发人员调试程序使用。
    
- **INFO**： 记录程序运行过程中的关键信息（如运维数据）。
    
- **WARN**： 记录警告信息，表明可能有问题，但不影响系统运行。
    
- **ERROR**： 记录错误信息，会影响程序运行。
    
- **FATAL**： 灾难性错误，通常合并到 ERROR 级别中。
    

**规律**：日志级别具有“向上继承”的特性。例如，当你设置日志级别为 `WARN`时，只有 **WARN**和 **ERROR**级别的日志会被输出，而 **DEBUG**和 **INFO**级别的日志将被忽略。

### 2. 代码中使用日志
#### 以前的做法
一个标准的在 Controller 中使用日志的示例：

```
@RestController
@RequestMapping("/books")
public class BookController {
    // ①：创建日志记录器对象
    private static final Logger log = LoggerFactory.getLogger(BookController.class);

    @GetMapping
    public String getById() {
        // 使用不同级别记录日志
        log.debug("debug ..."); // 调试信息
        log.info("info ...");   // 常规信息
        log.warn("warn ...");   // 警告信息
        log.error("error ..."); // 错误信息
        return "springboot is running...";
    }
}
```

**关键点**：

- 应使用 `log.debug/info/...()`替代 `System.out.println()`，因为后者不能控制输出级别，且性能较差。
    
- 通过注入不同的日志记录器（如 Logback），可以实现丰富的输出格式和目的地（控制台、文件等）。
#### 推荐做法（lombok的 `@Slf4j`）
图片中的代码演示了如何通过 Lombok 工具的 `@Slf4j`注解，便捷地在控制器（Controller）中记录日志。

**1. 代码要点说明：**

- **注解作用**：`@Slf4j`是 Lombok 提供的一个注解。它的最大优点是**简化开发**。在编译时，Lombok 会自动为这个类生成一个名为 `log`的日志对象（Logger），开发者无需再手动编写 `private static final Logger log = LoggerFactory.getLogger(BookController.class);`这行代码。
    
- **功能**：在 `getById`方法中，使用自动注入的 `log`对象记录了不同级别（DEBUG, INFO, WARN, ERROR）的日志信息。


```
@Slf4j // 注解在此
@RestController
@RequestMapping("/books")
public class BookController {
    @GetMapping
    public String getById(){
        System.out.println("springboot is running...");
        // 使用注解自动生成的 log 对象，首字母为小写
        log.debug("debug info...");   // 修正：log.debug(...)
        log.info("info info...");     // 修正：log.info(...)
        log.warn("warn info...");     // 修正：log.warn(...)
        log.error("error info...");   // 修正：log.error(...)
        return "springboot is running...";
    }
}
```

### 3. 配置日志
#### 配置日志输出级别
有两种主要方式：

##### **① 简单全局设置**

在 `application.yml`中配置，设置整个应用的默认日志级别为 `debug`：

```
# 旧式配置（但仍可用）
debug: true
# 或推荐的标准配置方式
logging:
  level:
    root: debug  # root 表示根记录器，即所有日志
```

##### ② 精细分组设置（group）

这是更强大和实用的配置方式，可以为不同的包或自定义组设置不同的日志级别。

```
logging:
  group:
    # 定义一个名为 'ebank' 的组，包含指定包
    ebank: com.itheima.controller
  level:
    root: warn           # 全局默认级别为 warn（只显示警告和错误）
    ebank: debug         # 但 'ebank' 组（即controller包）的级别为 debug（显示详细调试信息）
    com.itheima.controller: debug # 也可以不分组，直接指定包
```

**配置的好处**：

- **环境适配**：在开发环境设置为 `DEBUG`便于排查问题，在生产环境设置为 `WARN`或 `ERROR`以减少日志量、提升性能并保护敏感信息。
    
- **聚焦问题**：当排查某个模块的问题时，可以仅将该模块（包）的日志级别调低（如 `DEBUG`），而其他模块保持较高级别（如 `WARN`），避免日志泛滥。
    


#### 配置日志的样式


##### 1. 默认日志格式解析

**默认日志格式**：每一条日志都包含以下几个关键部分，构成了一个非常信息化的标准结构：

- **时间**(`%d`)：日志记录的时间戳，精确到毫秒。例如：`2021-11-02 12:25:39.392`。
    
- **日志级别**(`%p`)：如 `INFO`, `DEBUG`, `ERROR`等。它可以帮助你快速筛选和关注特定级别的信息（如所有错误）。
    
- **进程ID (PID)**：当前 Java 应用程序的进程编号。例如：`2336`。**作用**：当服务器上同时运行多个 Spring Boot 应用时，PID 可以帮助你清晰地区分不同应用产生的日志，对调试至关重要。
    
- **线程名**(`%t`)：执行当前代码的线程名称。例如：`[main]`。**作用**：对于诊断多线程环境下的问题非常有帮助。
    
- **日志记录器名**(`%c`)：输出日志的类或接口的全限定名。**特点**：Spring Boot 对过长的类名进行了优化显示，会进行缩写以保持日志行的简洁。
    
- **日志消息**(`%m`)：这是开发者实际调用日志方法（如 `log.info("...")`）时记录的具体信息。
    

**总结**：默认格式提供了非常全面的上下文信息，非常适合开发和测试环境使用。

##### 2. 自定义日志输出格式

**基本语法**：

在 `application.yml`中，使用 `logging.pattern.console`属性来定义控制台日志的格式。

**示例一：极简格式**

```
logging:
  pattern:
    console: "%d - %m%n"
```

- `%d`： 输出日期时间。
    
- `%m`： 输出原始日志消息。
    
- `%n`： 代表换行符。
    
- **输出效果**：`2021-11-02 12:25:39.392 - This is a log message`
    

**示例二：复杂且带颜色的格式**

```
logging:
  pattern:
    console: "%d %clr(%p) --- [%t] %clr(%c){cyan} : %m %n"
```

这个示例更复杂，功能也更强大：

- `%clr(...)`： 这是 Spring Boot 的特色功能，用于为内容着色。括号内的部分会显示为彩色。
    
    - `%clr(%p)`： 日志级别会根据其重要性自动显示为不同的颜色（如 ERROR 为红色，INFO 为绿色）。
        
    
- `---`： 普通的分隔符。
    
- `[%t]`： 输出线程名，并用方括号括起来。
    
- `%clr(%c){cyan}`： 输出日志记录器名（类名），并强制指定其颜色为青色 (`cyan`)。
    
- **输出效果**：一个带有高亮和颜色的、结构清晰的日志行，可读性更强。

- `%d`： 日期时间

- `%p`： 日志级别

- `%m`： 日志消息

- `%n`： 换行

- `%t`： 线程名

- `%c`： 日志记录器名（类名）

- `%clr(...)`： 颜色渲染
#### 配置日志的输出文件

- **基础方法**：使用 `logging.file.name`将日志保存到文件。
    
- **最佳实践**：使用 **滚动归档策略**，通过配置 `max-file-size`和 `file-name-pattern`：
    - **避免单个文件无限增大**。
    - **自动按大小和日期对历史日志进行归档**，便于管理和查阅（例如，按天进行问题排查）。
    - **有效利用磁盘空间**。

在实际应用中，你只需要将 `max-file-size`调整为一个合理的值（如 `10MB`）即可。
##### 1. 基础配置：设置日志文件

这是最简单的配置，目的是将日志从控制台输出重定向到指定的文件。

```
logging:
  file:
    name: server.log
```

- **作用**：应用启动后，所有日志将不再仅打印在控制台，而是会**同时**被写入到项目根目录下的 `server.log`文件中。（在IDEA中看不到，要去系统文件管理器看）
    
- **特点**：这是一个单一的日志文件。随着应用运行，该文件会变得越来越大，不利于管理和排查问题。
    

##### 2. 高级配置：日志滚动与归档

这是更专业和推荐的配置方式，它解决了单个日志文件过大的问题。

```
logging:
  file:
    name: server.log
  logback:
    rollingpolicy:
      max-file-size: 3KB
      file-name-pattern: server.%d{yyyy-MM-dd}.%i.log
```

这个配置实现了“滚动日志”策略，其工作逻辑如下：

- **`logging.file.name: server.log`**：这个文件被指定为**当前正在写入的日志文件**（也称为活跃文件）。
    
- **`max-file-size: 3KB`**：设置单个日志文件的**最大容量**。示例中为了演示设置为 3KB，实际生产中通常会设置为 10MB、100MB 等。当 `server.log`文件的大小达到 3KB 时，就会触发滚动操作。
    
- **`file-name-pattern: server.%d{yyyy-MM-dd}.%i.log`**：定义了归档日志文件的**命名规则**。它包含两个重要占位符：
    
    - **`%d{yyyy-MM-dd}`**：归档发生的日期。
        
    - **`%i`**：一个递增的序号，用于解决同一天内产生多个归档文件的问题。
        
    

**工作流程举例**：

1. 应用启动，开始向 `server.log`写入日志。
    
2. 当 `server.log`文件大小达到 3KB 时：
    
    - 当前的 `server.log`会被**重命名**（归档）为一个新文件，例如 `server.2024-05-27.0.log`。
        
    - 然后，系统会立即创建一个新的、空的 `server.log`文件来接收后续的日志。
        
    
3. 如果当天日志量很大，`server.log`再次达到 3KB，会再次归档为 `server.2024-05-27.1.log`，以此类推。

