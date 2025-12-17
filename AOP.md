---
aliases:
  - 面向切面编程
---
- 需求背景： **交叉业务代码泛滥**![[Pasted image 20251108095313.png]]
	- **现象**：日志、事务、安全等“交叉业务”代码混杂在核心业务代码（如订单处理、用户管理）中。
	- **后果**：
		- **代码复用性差**：相同的日志记录代码出现在无数个方法中。
		- **难维护**：修改日志格式需要改动大量文件。
		- **专注度分散**：程序员需要同时关心业务逻辑和技术细节。
- 解决方案：AOP
	- **核心思想**：将交叉业务代码（如日志）从核心业务代码中**剥离出来**，独立设计和开发。
		- 我们设计一个个切面类，通过切点表达式匹配切点，然后通过注解设定连接点，将我们的通知根据指定的顺序织入，从而实现将交叉业务代码（如日志）从核心业务代码中**剥离出来**，独立设计和开发。
    - **实现方式**：通过**动态代理技术**，在程序运行期间，将剥离出来的交叉业务代码**动态地“织入”** 到核心业务代码的特定位置。
    - **最终效果**：代码结构清晰，核心业务模块高度内聚，交叉业务模块得以复用。
        ![[Pasted image 20251108101151.png]]
		- 通用的**交叉业务**代码（日志记录，安全，事务管理）得到复用
		- 通用的**交叉业务**代码与业务代码分离，便于维护
- AOP 的核心概念![[Pasted image 20251108100803.png]]
	- 通知：起增强作用的代码
		- 匹配切点的方式：切点表达式
	- 切点：要被通知增强的方法
		- 一个切点对应多个连接点。
	- 连接点：通知可以织入的地方
		- 程序执行过程中（如方法调用、异常抛出）可以插入切面的**所有可能的位置**。
	- 切面：切点+通知（类似于[[InvocationHandler]]的invoke()方法的代码）
		- 定义了“在何处（切点）”执行**“**什么**（通知）
- 问题：AOP失效场景：
	- 最常见的情况：同一个类中的非AOP方法调用AOP方法。
	- 原因：这种调用绕过了代理对象，直接调用了目标类自身的方法，导致切面无法介入。
	- 解决措施：通过 `AopContext.currentProxy()`获取当前代理对象再进行调用
- Spirng Boot中使用：
	1. 引入依赖
		```
		<dependency>
		    <groupId>org.springframework.boot</groupId>
		    <artifactId>spring-boot-starter-aop</artifactId>
		</dependency>
		```
	2. 创建切面类
		``` package com.example.demo.aspect;
		
		import org.aspectj.lang.annotation.*;
		import org.springframework.stereotype.Component;
		
		@Aspect
		@Component
		public class LoggingAspect {
		    // 切点定义将在此处
		    // 通知方法将在此处定义
		}
		```
		- `@Aspect`注解：标记它为一个切面
		- `@Component`注解：将其交由 Spring 容器管理
	3. 切面类中切点:
		```
		    // 匹配 com.example.demo.service 包下所有类的所有方法
		    @Pointcut("execution(* com.example.demo.service.*.*(..))")
		    public void serviceMethods() {} // 此方法体通常为空，仅作为切入点标识
		 
		```
		- 使用 `@Pointcut`注解定义在哪些方法上执行通知：[[切点表达式]]
		- 切点用于通知方法中复用
	4. 切面类中编写通知
		- [[AOP五种通知类型]]
		- [[多切面执行顺序]]
		- 在通知中获得当前正在执行的目标方法的信息[[joinpoint]]
	
	