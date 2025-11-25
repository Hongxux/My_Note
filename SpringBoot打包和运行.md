---
aliases:
  - 打包与运行
---
**Spring Boot 项目从源代码打包成可执行的独立 Jar 包，并在 Windows 和 Linux 系统上快速启动运行**的全过程、原理和注意事项。

### 一、 核心目标：打包与快速启动

这组教程的最终目标是：**将 Spring Boot 项目变成一个随处可运行的、独立的自包含（self-contained）应用**。

**核心价值**：你可以在任何有 Java 环境的机器上，无需安装外部 Web 服务器（如 Tomcat），直接通过一条命令 `java -jar yourapp.jar`启动一个完整的 Web 应用。

### 二、 实现原理：Spring Boot Maven 插件

这是实现上述目标的**技术基石**。

1. **作用**：**`spring-boot-maven-plugin`插件的核心作用就是“打出独立运行的 Jar 包”**。它不是一个普通的 Jar 包，而是一个“超级”Jar。
    
2. **配置**：在项目的 `pom.xml`中必须配置此插件。
    
    ```
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
    ```
    
1. **打包命令**：在项目根目录下执行 Maven 命令。
    
    ```
    mvn package
    ```
    

### 三、 可执行 Jar 包的奥秘

普通 Jar 包和 Spring Boot 可执行 Jar 包的关键区别在于**内部结构**和**启动方式**。

#### 1. 特殊的目录结构

打包后生成的 Jar 包内部有一个 **`BOOT-INF`** 目录，这是 Spring Boot Jar 的标识。

- **`BOOT-INF/classes/`**：存放你项目编译后的所有 **`.class`** 文件和配置文件（如 `application.yml`）。
    
- **`BOOT-INF/lib/`**：存放项目运行所需的**所有依赖 Jar 包**。正是这个设计，使得最终 Jar 包是“完整”的。
    

#### 2. 核心描述文件：`MANIFEST.MF`（图4）

这个文件位于 `META-INF/`下，它告诉 Java 虚拟机如何启动这个 Jar。

- **普通 Jar 包**的 `Main-Class`直接指向你自己的启动类（如 `SSMPApplication`）。
    
- **Spring Boot 可执行 Jar 包**的 `Main-Class`指向的是 **`org.springframework.boot.loader.JarLauncher`**。这是一个由 Spring Boot 提供的**引导程序**。
    

**`JarLauncher`的工作流程**（对应图表）：

1. **启动**：当你执行 `java -jar springboot.jar`时，JVM 根据 `MANIFEST.MF`找到并执行 `JarLauncher`的 `main`方法。
    
2. **构建类路径**：`JarLauncher`这个“启动器”会读取 `BOOT-INF/lib`下的所有依赖 Jar 包和 `BOOT-INF/classes`下的应用类，**构建一个特殊的类加载路径**。
    
3. **委托启动**：最后，`JarLauncher`再根据 `MANIFEST.MF`中指定的 `Start-Class`（即你自己的 `SSMPApplication`类），反射调用其 `main`方法，从而启动整个 Spring Boot 应用。

### 四、 启动操作指南

#### 1. Windows 系统启动

打包成功后，在 Jar 包所在目录打开命令行，执行：

```
java -jar your-application.jar
```

#### 2. Linux 系统启动
流程与 Windows 完全一致，但需要注意：

- **JDK 版本**：Linux 服务器上安装的 JDK 版本**不能低于**你打包时使用的 JDK 版本。
    
- **安装路径**：建议将 Jar 包放在 `/usr/local/yourapp/`或用户目录 `$HOME`下。
    

### 五、 常见问题与解决方案--**端口被占用**

在 Windows 上启动时，最常遇到的问题就是**端口被占用**。

解决方案是一套完整的进程查杀流程：

```
# 1. 查询端口占用情况（例如查询8080端口）
netstat -ano | findstr "8080"
# 输出示例: TCP 0.0.0.0:8080 0.0.0.0:0 LISTENING 12345

# 2. 根据上一步查到的PID（12345），查询是哪个进程
tasklist | findstr "12345"
# 输出示例: java.exe 12345 Console 1 1,000,000 K

# 3. 强制终止该进程
taskkill /F /PID 12345
# 或者根据进程名终止
taskkill -f -t -im "java.exe"
```