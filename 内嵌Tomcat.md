![[Pasted image 20251110215555.png]]
**“内嵌Tomcat服务器是SpringBoot辅助功能之一”**
**Spring Boot 的变革**：它将服务器（如 Tomcat、Jetty、Undertow）**直接打包到您的应用中**，成为一个**可执行的 JAR 文件**。您无需预装任何 Web 服务器，直接通过 `java -jar yourapp.jar`命令即可启动一个完整的 Web 应用。

**价值**：这极大地简化了**开发、部署和运维**的流程，实现了真正的**独立应用**。


##### 原理：将服务器作为被管理的对象
如何理解内置服务器：
	tomcat是java写的，运行的本质是对象，而内置tomcat服务器，就是相当于把Tomcat服务器这个对象交给spring框架管理。

这是技术上最核心、最巧妙的一点。Spring Boot 颠覆了传统认知，**不再是应用部署在服务器里，而是服务器作为应用的一部分，被Spring容器所管理**。

**工作原理详解**：

1. **依赖引入**：当您在 `pom.xml`中引入 `spring-boot-starter-web`时，它会传递性地引入 `spring-boot-starter-tomcat`，从而将 Tomcat 的 JAR 包添加到您的类路径下。
    
2. **对象化**：Spring Boot 在启动时，会通过**Java代码**（而非XML配置）来**实例化**一个 Tomcat 对象。
    
    ```
    // 概念性代码，Spring Boot 内部大致如此处理
    Tomcat tomcat = new Tomcat();
    tomcat.setPort(8080);
    // ... 其他配置
    ```
    
3. **纳入IoC容器管理**：这个被实例化的 Tomcat 对象，会被**注册为 Spring 容器中的一个 Bean**。这意味着：
    
    - 它的**生命周期**（启动、停止）由 Spring 容器管理。当 Spring 应用上下文启动时，Tomcat 也随之启动；当上下文关闭时，Tomcat 也优雅关闭。
        
    - 它可以**轻松地被注入**到其他 Bean 中，以便进行自定义配置。
        
    - 它的配置（如端口、上下文路径）可以通过 Spring 的配置机制（`application.properties`）进行**外部化配置**，高度灵活。
        
    

##### 扩展：灵活的服务器切换策略

> **“变更内嵌服务器思想是去除现有服务器，添加全新的服务器”**

这体现了 Spring Boot 设计的**灵活性**和**解耦**思想。因为服务器是以**依赖**的形式引入的，所以替换它变得非常简单。

**具体操作示例（将内嵌服务器从 Tomcat 切换到 Jetty）**：

在 Maven 的 `pom.xml`中：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- 1. 排除默认的 Tomcat -->
        <exclusions>
            <exclusion>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-tomcat</artifactId>
            </exclusion>
        </exclusions>
    </dependency>
    
    <!-- 2. 添加想要的 Jetty 服务器 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jetty</artifactId>
    </dependency>
</dependencies>
```

**切换原理**：

- Spring Boot 的自动配置模块中为不同的服务器（Tomcat, Jetty, Undertow）都提供了配置类。
    
- 这些配置类上标有 `@ConditionalOnClass`注解，例如：`@ConditionalOnClass({Tomcat.class, UpgradeProtocol.class})`。
    
- 当您排除 Tomcat 并引入 Jetty 后，类路径下就不存在 `Tomcat.class`，而是存在 `Jetty.class`。于是，Tomcat 的自动配置失效，Jetty 的自动配置生效，Spring Boot 便会自动为您创建并管理一个 Jetty 服务器实例。
    

**为什么要切换？**

不同的内嵌服务器在**性能**（如并发处理、内存占用）、**资源消耗**和**特性**（如WebSocket支持）上各有优劣，开发者可以根据应用场景选择最合适的服务器。