---
aliases:
  - 多环境配置
---
![[Pasted image 20251111201551.png]]
在软件开发中，通常会有多个环境，例如：

- **开发环境 (dev)**：开发人员本地调试使用。
    
- **测试环境 (test)**：测试人员进行功能测试。
    
- **生产环境 (prod)**：最终用户使用的线上环境。
    

每个环境的配置参数（如数据库地址、日志级别、服务器端口等）通常不同。Spring Boot 的 **Profile**机制就是为了**将不同环境的配置隔离**，并能轻松切换。

---

### 一、 配置方法：
#### 在 YAML 文件中定义多环境

图1和图2的核心是展示如何在一个配置文件（如 `application.yml`）中，使用 `---`分隔符来定义多个环境配置。

##### 1. 基础语法结构

```
# 应用通用配置（所有环境共享）
spring:
  profiles:
    active: pro # 指定默认激活的环境

---
# 生产环境配置
spring:
  profiles: pro # 定义此段配置的环境名为 pro
server:
  port: 80      # 生产环境的端口

---
# 开发环境配置
spring:
  profiles: dev # 定义此段配置的环境名为 dev
server:
  port: 81      # 开发环境的端口

---
# 测试环境配置
spring:
  profiles: test # 定义此段配置的环境名为 test
server:
  port: 82      # 测试环境的端口
```

##### 2. 语法详解

- **`---`**（三个减号）：这是 YAML 语言中定义**文档边界**的符号。在 Spring Boot 中，它被用来**分隔不同环境的配置块**。每个 `---`之后开始一个新的配置段。
    
- **`spring.profiles`**：在每个配置段中，使用 `spring.profiles`来**为此段配置命名**，即定义环境标识（如 `pro`, `dev`, `test`）。
    
- **环境特定配置**：在每个命名后的配置段中，编写该环境下**特有的配置**。如上例中，每个环境的 `server.port`都不同。

#### 多文件中定义多环境
![[Pasted image 20251111202257.png]]

##### 1. 配置文件结构

在 `src/main/resources`目录下，你会看到一组配置文件：

- **`application.yml`**：**主配置文件**。它不包含具体环境参数，只负责**激活**哪个环境。
    
- **`application-dev.yml`**：**开发环境**的专用配置。
    
- **`application-pro.yml`**：**生产环境**的专用配置。
    
- **`application-test.yml`**：**测试环境**的专用配置。
    

##### 2. 配置内容详解

**主配置文件 (`application.yml`)**

```
# 这个文件唯一重要的作用是指定当前激活哪个环境
spring:
  profiles:
    active: dev  # 激活开发环境。可改为 pro, test 等。
```

**环境配置文件 (以 `application-dev.yml`为例)**

每个环境配置文件只包含该环境特有的、会与其他环境产生冲突的配置。

```
# 开发环境配置
server:
  port: 81  # 开发环境使用 81 端口

# 可以继续配置开发环境的数据库、日志级别等
```

##### 3. 工作原理

1. Spring Boot 启动时，首先加载 `application.yml`。
    
2. 根据 `spring.profiles.active`的设置（例如 `dev`），Spring Boot 会**额外加载**对应的环境配置文件（`application-dev.yml`）。
    
3. 环境配置文件中的属性**覆盖**或**补充**主配置文件中可能存在的同名属性。
    
4. 最终，应用使用**合并后**的配置信息启动。
    

**示例**：当 `spring.profiles.active: pro`时，会合并 `application.yml`和 `application-pro.yml`的配置，最终 `server.port`为 `81`。


---

### 二、 激活环境：如何指定使用哪个环境？

配置了多个环境后，你需要告诉 Spring Boot 当前要使用哪一个。图1中 `spring.profiles.active: pro`就演示了这一点。

#### 激活方式（从高到低优先级）

1. **在配置文件中指定（默认激活）**：
    
    如图1所示，在通用配置部分使用 `spring.profiles.active`设置。这是**最低优先级**的，通常用于设置默认环境。
    
    ```
    spring:
      profiles:
        active: dev # 默认使用开发环境
    ```
    
2. **命令行激活（最常用、优先级最高）**：
    
    在启动应用时，通过 `--`参数临时指定环境，这会**覆盖配置文件中的设置**。
    
    ```
    # 激活生产环境
    java -jar your-app.jar --spring.profiles.active=pro
    # 激活测试环境
    java -jar your-app.jar --spring.profiles.active=test
    ```
    
3. **IDE 中激活（开发阶段便捷）**：
    
    在 IDEA 等开发工具的运行配置中，通过 `Program arguments`指定。
    
    ```
    --spring.profiles.active=dev
    ```
    
4. **系统环境变量/虚拟机参数激活**：
    
    ```
    # 设置为系统环境变量
    export SPRING_PROFILES_ACTIVE=pro
    # 或作为JVM参数启动
    java -Dspring.profiles.active=pro -jar your-app.jar
    ```
    

**优先级规则**：**命令行参数 > 系统环境变量 > 配置文件设置**。这意味着通过命令行可以轻松覆盖默认配置，非常灵活。
#### maven中控制SpringBoot开发环境
maven配置比SpringBoot配置更具有主导权
![[Pasted image 20251111204716.png]]
### 三、多环境配置书写技巧

#### 原则一：主配置设置公共配置（全局）

- **做法**：将所有环境中**相同的配置**（即**不随环境变化**的配置）写在 `application.yml`中。
    
- **示例**：
    
    ```
    # application.yml - 全局配置
    spring:
      profiles:
        active: dev # 激活的环境（这个本身也是环境相关的，是特例）
    myapp:
      name: MySpringBootApp # 应用名称，所有环境都一样
      version: 1.0.0        # 应用版本，所有环境都一样
    ```
    

#### 原则二：环境配置设置冲突属性（局部）

- **做法**：将那些**每个环境都可能不同**的配置，分别写入对应的环境配置文件中。
    
- **常见的“冲突属性”包括**：
    
    - **服务器端口**(`server.port`)
        
    - **数据库连接**(`spring.datasource.url`, `username`, `password`)
        
    - **日志级别**(`logging.level`)
        
    - **第三方服务密钥**等。
        
    

**示例：环境配置文件内容**

```
# application-dev.yml - 开发环境局部配置
server:
  port: 8081
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db
    username: dev_user
    password: dev_pass
logging:
  level:
    com.example: debug # 开发环境开启详细调试日志
```

```
# application-pro.yml - 生产环境局部配置
server:
  port: 80
spring:
  datasource:
    url: jdbc:mysql://prod-db.example.com:3306/prod_db
    username: prod_user
    password: ${DB_PASSWORD} # 生产环境密码通常用环境变量注入，更安全
logging:
  level:
    com.example: warn # 生产环境只记录警告和错误日志
```
好的，这三张图片系统性地阐述了在 Spring Boot 中，为了应对复杂配置而采用的**配置拆分、组合加载**的高级特性，以及随着版本迭代而产生的**最佳实践演变**。

下面我将为您整合这三张图的信息，提供一个完整的解析。

---

#### 原则三：配置模块化+分组管理


##### 一、 配置的拆分与组合

图1展示了配置管理的两个核心动作：**拆分**和 **组合**。

###### 1. 拆分：按功能模块化

**目标**：将一个大的 `application-dev.yml`文件，按技术栈或功能拆分成多个独立的、更专注于特定领域的配置文件。

**拆分示例**：

- **`application-devDB.yml`**：只包含开发环境的数据源（DataSource）配置。
    
- **`application-devRedis.yml`**：只包含开发环境的 Redis 连接配置。
    
- **`application-devMVC.yml`**：只包含开发环境的 Web MVC 相关配置。
    

**好处**：配置结构清晰，便于管理。例如，数据库管理员只需关心 `-DB.yml`文件。

###### 2. 组合：使用 `include`属性（旧版方式）

**目标**：在激活主环境（如 `dev`）时，**同时激活**多个拆分后的子配置。

**配置方式**：

```
spring:
  profiles:
    active: dev       # 1. 激活主环境'dev'
    include: devDB, devRedis, devMVC # 2. 包含三个子配置
```

**效果**：应用启动后，会加载 `application-dev.yml`, `application-devDB.yml`, `application-devRedis.yml`, `application-devMVC.yml`这四个文件的所有配置。

---

##### 二、 配置属性的优先级规则

图2是**非常关键**的提示，它明确了当多个配置文件包含相同属性时，**哪个配置会最终生效**的冲突解决规则。

**规则一：主环境配置优先**

> 当主环境（如 `dev`）与其他环境（如 `devDB`）有相同属性时，**主环境（`dev`）的属性生效**。

- **示例**：如果 `application-dev.yml`和 `application-devDB.yml`都设置了 `spring.datasource.url`，那么最终会使用 `application-dev.yml`中的值。
    

**规则二：后加载者优先（对于同级别配置）**

> 在 `include`列表中的多个配置里，有相同属性时，**最后被加载的配置属性生效**。

- **示例**：在 `include: devDB, devRedis, devMVC`中，如果 `devRedis.yml`和 `devMVC.yml`都设置了 `server.port`，那么最终会使用 `devMVC.yml`中的设置，因为 `devMVC`在列表中最后被加载。
    

**记忆技巧**：你可以将其想象为“**后来者居上**”，最后的配置会覆盖前面的。

---

##### 三、 配置方式的演进：从 `include`到 `group`

图3介绍了 Spring Boot 2.4 版本引入的**更先进、更强大的 `group`属性**，它旨在替代 `include`。

###### 1. 为什么需要 `group`？

- **降低配置书写量**：`include`的配置是全局的，与 `active`紧耦合。而 `group`允许你为**每个主环境**预定义其所需包含的子环境组。
    
- **逻辑更清晰**：将“主环境”与它所依赖的“子环境”捆绑在一起，形成一个个**环境组**，管理起来更直观。
    

###### 2. `group`的使用方式

```
spring:
  profiles:
    active: dev  # 激活主环境'dev'
    group:
      "dev": devDB, devRedis, devMVC   # 定义‘dev’组包含哪些子配置
      "pro": proDB, proRedis, proMVC   # 定义‘pro’组
      "test": testDB, testRedis, testMVC # 定义‘test’组
```

**工作流程**：

1. 应用启动，读取到 `active: dev`。
    
2. Spring Boot 查找 `group`下名为 `"dev"`的分组。
    
3. 找到该组定义的子配置列表 `devDB, devRedis, devMVC`。
    
4. **按顺序加载**：`application-dev.yml`-> `application-devDB.yml`-> `application-devRedis.yml`-> `application-devMVC.yml`。
    
5. **优先级规则**与使用 `include`时**完全一致**（主环境优先，同级别后加载优先）。
    

###### 3. `group`的优势

- **配置集中化**：所有环境的组合关系在一个地方定义，一目了然。
    
- **切换方便**：只需修改 `active`的值（如改为 `pro`），就会自动切换到对应的 `"pro"`组，无需修改 `include`列表。
    
- **避免错误**：防止了在切换主环境时忘记同步修改 `include`列表而导致的配置错误。
    
