---
aliases:
  - Starter
  - 依赖组合模块
---


![[Pasted image 20251110213112.png]]
Starter 的本质是一个 **“依赖包”或“功能模块包”** 。它将完成某个特定功能所需的所有 **依赖**、**配置**（通过自动配置）打包在一起，提供给开发者。

**使用 Starter 的流程**：

1. **选择**：根据所需功能，在 `pom.xml`中添加对应的 `spring-boot-starter-*`。
    
2. **使用**：直接开始编码，Spring Boot 已经为你准备好了运行环境。
    
3. **定制**：如有需要，在 `application.properties`或 `application.yml`中修改默认配置。
    

**举例**：如果你想创建一个提供 REST API 的 Web 项目，只需要：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

然后编写一个主类和 Controller，即可启动应用。Starter 机制帮你处理了所有背后的复杂依赖和初始配置。

##### 一、 实例分析：`spring-boot-starter-web`的构成

第一张图展示了官方提供的 `spring-boot-starter-web`启动器的内部结构。这是一个非常经典的例子。

**1. 它是一个依赖集合（聚合器）**

这个 `spring-boot-starter-web.pom`文件本身不包含代码，它只是一个**依赖声明列表**。当你引入它时，Maven/Gradle 会自动引入所有列出的依赖：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId> <!-- 1. 内嵌Tomcat服务器 -->
        <version>2.5.4</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-web</artifactId> <!-- 2. Spring Web 核心模块 -->
        <version>5.3.9</version>
    </dependency>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId> <!-- 3. Spring MVC 框架 -->
        <version>5.3.9</version>
    </dependency>
    <!-- ... 通常还会包含json处理、验证等相关依赖 -->
</dependencies>
```

**解读**：这意味着，你只需要在项目中添加一个 `spring-boot-starter-web`依赖，就相当于自动引入了**开发一个Web应用所需的所有标准组件**。无需再手动、逐个地去添加Tomcat、Spring MVC等依赖，且版本都是匹配好的。

**2. 传递性依赖与版本管理**

图中右侧展示了 `spring-boot-starter-tomcat`会进一步引入内嵌Tomcat所需要的所有组件（如 `tomcat-embed-core`, `tomcat-embed-el`等）。这一切对开发者都是**透明**的，版本也由 Spring Boot 父工程统一管理，避免了依赖冲突。

##### 二、 理论总结：Starter 的核心价值

第二张图从理论层面精炼地总结了 Starter 的三个核心价值，完美诠释了第一张图展示的实例。

**1. 简化依赖导入**

> “开发 Spring Boot 程序需要导入坐标时通常导入对应的 starter”

这是 Starter 最直接的作用。它遵循了**约定大于配置**的原则。你需要使用某项技术时，不再需要记忆和查找一堆零散的依赖坐标，只需引入对应的 Starter 即可。

- **传统方式**：想用 JPA，需要手动添加 `hibernate-core`, `spring-orm`, 数据源等多个依赖，并确保版本兼容。
    
- **Spring Boot 方式**：只需添加一个 `spring-boot-starter-data-jpa`。
    

**2. 功能依赖聚合**

> “每个不同的 starter 根据功能不同，通常包含多个依赖坐标”

这正是第一张图所展示的。Starter 是按**功能**组织的，而不是按**技术**组织的。

- `spring-boot-starter-web`：聚合了**Web开发**所需的所有依赖。
    
- `spring-boot-starter-data-jpa`：聚合了**JPA数据访问**所需的所有依赖。
    
- `spring-boot-starter-test`：聚合了**测试**所需的所有依赖（JUnit, Mockito, Spring Test等）。
    

**3. 实现快速配置**

> “使用 starter 可以实现快速配置的效果，达到简化配置的目的”

这是 Starter 更深层次的价值，它常常与 **自动配置（Auto-Configuration）**机制协同工作。

- **依赖传递**：Starter 引入了必要的库。
    
- **自动配置**：当 Spring Boot 在类路径下检测到这些库时（例如，因为引入了 `spring-boot-starter-web`而检测到Spring MVC和内嵌Tomcat），它会**自动配置**这些组件所需的Bean（如 `DispatcherServlet`、`CharacterEncodingFilter`等），并提供一套合理的默认配置。
    

**结果**：开发者无需编写大量的 XML 或 Java 配置，一个基本的 Web 应用就可以直接运行。
