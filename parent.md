---
aliases:
  - Parent
  - 父POM集中管理
  - 版本统一管理
---

 ![[Pasted image 20251110182257.png]]
 通过 **“父POM集中管理”** 和 **“依赖组合模块”** 两级设计，解决了大型项目中依赖版本混乱和重复配置的痛点。实现**集中管理，一处修改，处处生效**


1. **统一版本控制**：避免不同子模块使用同一个依赖的不同版本，导致冲突。
    
2. **减少重复配置**：将常用的依赖组合打包，子模块直接引用，无需重复编写大量 ``。
    
3. **提升可维护性**：需要升级或更换依赖时，只需修改父POM或依赖组合模块，所有子项目会自动生效。

---

这张图指出了确保 Spring Boot 项目依赖和谐、避免版本冲突的基石——**`spring-boot-starter-parent`**。

##### 1. 核心起点：继承父工程

> **“开发SpringBoot程序要继承spring-boot-starter-parent”**

这是 Spring Boot 项目的**标准做法**。在你的 Maven 项目的 `pom.xml`中，第一步就是将其指定为父工程。

```xml
<!-- 你的项目的pom.xml -->
<project>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.1.0</version> <!-- 使用最新的稳定版本 -->
        <relativePath/> <!-- 从仓库查找，不直接从本地项目找 -->
    </parent>

    <groupId>com.example</groupId>
    <artifactId>my-spring-boot-app</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    ...
</project>
```

**作用**：这相当于让你的项目**继承**了一个预先配置好的、功能完善的 Maven 项目模板。

##### 2. 核心机制：依赖管理

> **“spring-boot-starter-parent中定义了若干个依赖管理”**

`spring-boot-starter-parent`本身继承了一个更基础的工程 `spring-boot-dependencies`。在这个基础工程中，包含了 Spring Boot 生态的 **“依赖账单”（Bill of Materials, BOM）**。

**BOM 的本质**：它是一个特殊的 `pom.xml`，其中在 `<properties>` 部分**集中定义了几百个常用依赖的版本号**。

**示例**（概念性展示）：
![[Pasted image 20251110212128.png]]
properites标签下管理着常用依赖的版本号![[Pasted image 20251110212149.png]]
dependencyManagement中锁定着一套映射关系，哪个依赖（K）对应着特定的版本（V），这些依赖的版本由properties标签指定。
	这样，当你在子项目中声明这些依赖时，**无需再指定版本号**，Maven 会自动使用 BOM 中锁定的版本。
![[Pasted image 20251110212328.png]]


```
<!-- 在spring-boot-dependencies的pom.xml中 -->
<properties>
    <spring-data-commons.version>3.1.0</spring-data-commons.version>
    <jackson.version>2.15.2</jackson.version>
    <mysql.version>8.0.33</mysql.version>
    <!-- ... 定义了海量的依赖版本 -->
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>${jackson.version}</version> <!-- 版本由上面的property控制 -->
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>
        <!-- ... 为每个依赖定义GAV坐标和版本 -->
    </dependencies>
</dependencyManagement>
```

##### 3. 核心价值：避免依赖版本冲突

> **“继承parent模块可以避免多个依赖使用相同技术时出现依赖版本冲突”**

这是使用 Parent 方式带来的**最大好处**。所谓“依赖版本冲突”，是指项目中间接引入了同一个JAR包的不同版本，导致Maven依据“最近原则”选择了可能不兼容的版本，从而引发 `ClassNotFoundException`、`NoSuchMethodError`等诡异问题。

**Spring Boot 的解决方案**：

通过统一的 BOM，Spring Boot 团队**已经替我们测试过**所有指定版本的兼容性。例如，他们确保 `spring-boot-starter-data-jpa 3.1.0`所引用的 `Hibernate`版本，与 `spring-boot-starter-web 3.1.0`完全兼容。

**你的收益**：

当你引入官方提供的 **Starter**时（如 `spring-boot-starter-web`），你完全无需关心它内部依赖了哪些库（如 Tomcat, Jackson, Spring MVC）以及它们的版本，因为所有版本都已被 Parent 统一管理。这被称为 **“开箱即用”** 的体验。

```
<dependencies>
    <!-- 直接引入Starter，无需版本号 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!-- 这两个Starter背后依赖的所有组件版本都是兼容
```