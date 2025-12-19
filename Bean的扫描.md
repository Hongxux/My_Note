---
aliases:
  - 组件扫描
---
​
    
- ​**必要性**​：仅仅在类上加了注解是没用的，​**必须被 `@ComponentScan`扫描到**，Spring 才会将其识别为 Bean 并纳入容器管理。
    

##### 2. Spring Boot 的简化：`@SpringBootApplication`

这是 Spring Boot 的核心便利性之一。

- ​**自动配置**​：虽然你没有显式地在代码中写下 `@ComponentScan`，但它已经被**隐式地包含**在了启动类（Application 类）的 `@SpringBootApplication`注解之中。
    
- ​**默认扫描规则**​：​**默认扫描启动类所在包及其所有子包**。
    

​**示例解读（结合图中目录结构）​**​：

假设你的启动类 `Application`在 `com.example.demo`包下：

```
package com.example.demo; // 启动类所在包

@SpringBootApplication // 此注解包含了 @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

那么，Spring 会自动扫描 `com.example.demo`以及其下的所有子包（如 `com.example.demo.controller`, `com.example.demo.service`等），并注册所有带注解的类。

​**重要提醒**​：如果你将 Controller、Service 等类创建在启动类所在包的**平级或上级包**中（例如放在 `com.example`下），​**它将不会被默认扫描到，导致 Bean 无法创建**。这时你需要手动配置 `@ComponentScan`来指定基础包。
