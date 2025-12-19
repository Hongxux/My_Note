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
    - **最终效果**：
        ![[Pasted image 20251108101151.png]]
		- 通用的**交叉业务**代码（日志记录，安全，事务管理）得到复用
		- 通用的**交叉业务**代码与业务代码分离，便于维护
- AOP 的核心概念![[Pasted image 20251108100803.png]]
	- 通知：起增强作用的代码
		- 匹配切点的方式：切点表达式
	- 切点：是匹配连接点的断言或表达式
		- 一个切点对应多个连接点。
	- 连接点：通知可以织入的地方
		- 程序执行过程中（如方法调用、异常抛出）可以插入切面的**所有可能的位置**。
	- 切面：切点+通知（类似于[[InvocationHandler]]的invoke()方法的代码）
		- 定义了“在何处（切点）”执行**“**什么**（通知）
- AOP失效场景：
	- 配置问题：
		- 未启用AOP支持（缺失`@EnableAspectJAutoProxy`）
		- 切面类或目标类未被Spring管理（缺失`@Component`等注解）
		- 组件扫描路径未包含切面类或目标类
	- 代理机制限制
		- 目标方法是`private`、`static`或`final`的
			- private、final的无法继承
			- static方法属于类，而AOP是基于对象代理
		- 同一个类中的非AOP方法调用AOP方法。
			- 原因：这种调用绕过了代理对象，直接调用了目标类自身的方法，导致切面无法介入。
			- 解决措施：
				- 方式一：获得代理对象，此操作因为涉及ThreadLocal，可能带来轻微的性能损耗
					1. 在配置类上添加`@EnableAspectJAutoProxy(exposeProxy = true)`
					2. 通过 `AopContext.currentProxy()`获取当前代理对象再进行调用
				- 方式二：自我注入：在类中注入自身的代理实例
				- 方式三：重构设计：将方法拆分到不同的类
	- 切点表达式问题：表达式书写错误，未匹配到目标方法
	- Bean作用域问题：目标Bean是多例（Prototype）且获取方式不当
		- 原因：
			- 单例Bean 属性注入只发生一次
			- 当单例Bean再熟悉注入的时候注入多例Bean，这个多例Bean就被固定，成了单例
			- AOP在代理这个多例Bean 的时候代理逻辑会执行，但最终都会委托给那个被固定下来的原始目标对象
		- 解决措施：使用 `ObjectProvide`
			- 作用：可以延迟地获取Bean实例
			- 使用方式：
				```java
				@Service // 单例
				public class SingletonService {
				    @Autowired
				    private ObjectProvider<PrototypeService> prototypeServiceProvider; // 注入Provider
				
				    public void doSomething() {
				        // 每次调用 getObject() 都会从容器获取一个新的 PrototypeService 实例（及其代理）
				        PrototypeService prototypeService = prototypeServiceProvider.getObject();
				        prototypeService.businessMethod();
				    }
				}
				```
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
		- [[AOP通知类型]]
		- 在通知中获得当前正在执行的目标方法的信息[[joinpoint]]
	
	