### 设计原理：
1. **技术集A（收集习惯）**：Spring Boot 团队预先收集了开发者常用的技术栈（如 Web、JDBC、Redis 等）。
    
2. **设置集B（收集设置）**：为这些技术栈收集了最常用的参数和默认值。
    
3. **初始化环境**：启动时，先加载用户自定义的 Bean 和依赖。
    
4. **定义与加载**：将技术集 A 中的所有技术都预先定义好对应的自动配置类（`XxxAutoConfiguration`）。
    
5. **条件化加载（核心）**：每个自动配置类上都有一系列 **`@ConditionalOn...`** 条件注解。Spring Boot 会将当前应用的环境（类路径、已有的 Bean、配置文件等）与这些条件进行比对，**只有件满足的配置类才会生效**。
    
6. **默认配置（约定）**：为生效的配置加载设置集 B 中的默认配置，极大减少开发者的配置工作量。
	加载默认配置的原理：[[基于外部配置的属性注入来驱动业务 Bean]]
    
7. **配置覆盖**：开放接口（主要是 `application.properties/yml`），允许开发者轻松覆盖任何默认配置。
---
### 源码步骤
### 1. 起点：`@SpringBootApplication`
![[Pasted image 20251113183956.png]]
你的启动类上会标注 `@SpringBootApplication`，它是一个复合注解，主要包含三个部分：
- `@SpringBootConfiguration`：标记这是一个 Spring 配置类。
- `@ComponentScan`：扫描当前包及其子包的 `@Component`、`@Service` 等组件。
- **`@EnableAutoConfiguration`**：**开启自动装配的关键**。
![[Pasted image 20251113184028.png]]
---

### 2. `@EnableAutoConfiguration` 做了什么？
![[Pasted image 20251113184050.png]]
这个注解通过 `@Import` 导入了 **`AutoConfigurationImportSelector`** 类。  
在 Spring 容器启动过程中，会调用 `AutoConfigurationImportSelector` 的 `selectImports` 方法，这个方法负责**决定要加载哪些自动配置类**。

---

### 3. 如何找到自动配置类？
![[Pasted image 20251113184244.png]]
![[Pasted image 20251113184416.png]]
在 `getAutoConfigurationEntry` 中，会调用 **`getCandidateConfigurations`**，这里使用 **`SpringFactoriesLoader.loadFactoryNames`** 从 **`META-INF/spring.factories`** 文件中读取所有声明的自动配置类（或新版 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`）
这些 `spring.factories` 文件分布在各个 Starter 的 jar 包中，比如 `spring‑boot‑autoconfigure‑x.x.x.jar` 中就定义了大量官方提供的自动配置类（如 `DataSourceAutoConfiguration`、`WebMvcAutoConfiguration` 等）。

在 Spring Boot 3 中，`AutoConfigurationImportSelector`内部的 `getCandidateConfigurations`方法不再从 `spring.factories`加载配置，而是转向读取新的 `.imports`文件
	这一变更使得 Spring Boot 3 的自动装配流程在支持 **[[GraalVM 原生镜像]]**上更具优势。GraalVM 的提前编译需要**静态分析代码**，而旧的 `spring.factories`机制依赖于运行时的类路径扫描，难以在构建时确定所有配置。新的 `.imports`文件以静态列表形式明确指出了所有候选配置类，便于 GraalVM 在编译时分析
**为何要改变？**

Spring Boot 团队做出这一变革主要基于以下几点考量：

1. **支持 GraalVM 原生镜像**：如前所述，这是最关键的驱动力。新的静态配置方式更适合 GraalVM 的编译模型。
    
2. **性能优化**：新的加载机制避免了合并和去重多个 `spring.factories`文件的操作，有助于提升应用启动速度。
    
3. **模块化支持**：新机制更好地兼容 Java 9 及以上版本的模块化系统。
    
4. **简化与清晰化**：`.imports`文件格式更简单易懂，降低了理解和维护成本。

---

### 4. 加载与过滤
![[Pasted image 20251113184512.png]]
并不是 `spring.factories` 里列出的所有配置类都会被加载。`AutoConfigurationImportSelector` 会执行以下步骤：

1. **获取候选配置类列表**（从 `spring.factories` 中读取）。
2. **去重**、**排除**（通过 `@EnableAutoConfiguration` 的 `exclude` 属性或配置 `spring.autoconfigure.exclude`）。
3. **条件判断**：利用 `@ConditionalOnClass`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty` 等条件注解，判断当前 classpath 下是否存在对应的类、是否已有 Bean 等，只有满足条件的配置类才会被真正加载。

例如，`DataSourceAutoConfiguration` 只有在 classpath 下有 `DataSource.class` 且你没有手动定义 `DataSource` Bean 时才会生效。

---

### 5. 配置类的内部机制
每个自动配置类本身是一个 `@Configuration` 类，里面通过 `@Bean` 方法声明需要注入的组件。  
例如 `WebMvcAutoConfiguration` 会在检测到 classpath 有 `Servlet`、`Spring MVC` 相关类时，自动配置 `DispatcherServlet`、视图解析器等 MVC 所需组件。

---

### 6. 自动装配的完整流程
简单总结一下流程：

1. **启动类** → `@SpringBootApplication` → `@EnableAutoConfiguration`  
2. **导入** `AutoConfigurationImportSelector`  
3. **读取所有jar包的**`META‑INF/spring.factories` 中的自动配置类列表 
4. **过滤**（去重、排除、条件判断）  
5. **加载符合条件的配置类** → 注册对应的 `@Bean` 到容器  
6. **完成自动装配**，你的项目直接拥有相应功能（如数据源、Web 服务器等）。

---

### 7. 为什么能做到“按需装配”？
关键在于 **条件注解** + **classpath 扫描**。  
Spring Boot 在编译时和启动时会检查你的依赖（classpath 中存在的 jar），只有当你引入了相关 Starter（比如 `spring‑boot‑starter‑web`），对应的自动配置类才会被激活。这就避免了不必要的 Bean 被加载，保证轻量与灵活。

---

### 小结
Spring Boot 的自动装配本质上是通过 **`@EnableAutoConfiguration`** → **`AutoConfigurationImportSelector`** → **`spring.factories`** → **条件注解** 这一链条，在容器启动时动态加载适合当前环境的配置类，从而让开发者几乎零配置就能使用大多数功能。  
如果你去看 `spring-boot-autoconfigure` 模块的源码，会发现大量以 `AutoConfiguration` 结尾的类，它们就是这套机制的具体实现。