### 准备空壳 
 名字：
 第三方的一般是 xxx-spring-boot-starter
 spirng 自己开发的一般是是spring-boot -starter-xxx

不要启动类和测试模块

配置文件改一个yml
### 组成和分工

#### 1. 一个业务类（service）
完成对应的功能，所需要的一些组件用autowired注入依赖
![[Pasted image 20251115191952.png]]
#### 2.一个autoconfig类
这是Starter的“大脑”，它决定在什么条件下创建你的业务功能Bean。
##### 1.负责引入Service
①import导入业务类
②@Bean
![[Pasted image 20251115192043.png]]
##### 2.负责引入属性配置封装类
![[Pasted image 20251115192944.png]]
#### 3.一个配置文件
位置
	在项目的 `src/main/resources`目录下创建文件：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
内容：autoconfig类
```
# 这是Spring Boot 2.7+ 推荐的方式
com.itheima.ip.autoconfigure.IpAutoConfiguration
```
作用：这是告诉Spring Boot“这里有一个自动配置类需要被加载”的方式。SpringBoot会在启动的时候扫描这个文件，将文件中提到的autoconfig类import（变成Bean）

为什么需要：引入我们starter的项目不会扫描我们的目录，而如果想要让我们的Bean被纳入管理，只能通过这样的方式

#### 4.属性配置封装类
[[基于外部配置的属性注入来驱动业务 Bean]]
[[@ConfigurationProperties]]
![[Pasted image 20251115193003.png]]
![[Pasted image 20251115192848.png]]
### jar包和导入jar包的项目的桥梁
共用一个Spring Ioc容器


### 使用
对这个项目先清除（clean）后安装（install），这样别的项目才可以依赖