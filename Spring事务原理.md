Spring事务基于AOP

当你在一个类的方法上使用 `@Transactional`注解时，Spring 在容器启动过程中会为该 Bean 创建**代理对象**。

- **代理模式**：你从 Spring 容器中注入的并不是原始的 Bean 实例，而是一个被增强过的代理对象。当你调用 `userService.updateUser()`时，实际上是在调用代理对象的方法。
    
- **拦截逻辑**：代理对象内部包含了事务拦截器（`TransactionInterceptor`）。它会在你真正的业务方法执行前后，自动完成事务管理的相关操作，如开启事务、提交事务或在异常时回滚事务。
	- 默认情况下，Spring 事务只在遇到**运行时异常**（`RuntimeException`及其子类）和 **Error**​ 时才会回滚。**受检异常**（Checked Exception）不会触发回滚。
		- 你可以通过 `@Transactional`注解的 `rollbackFor`/ `rollbackForClassName`属性来自定义需要回滚的异常类型
- **两种代理方式**：Spring 默认根据目标类是否实现了接口来选择使用 JDK 动态代理或 CGLIB 代理。CGLIB 代理可以代理未实现接口的类