- 事务的演变：
	1. 编程式事务（手动管理）：灵活但繁琐，需要在代码中显式控制事务边界。
		```
		@Autowired
		private TransactionTemplate transactionTemplate;
		
		public void manualTransaction() {
		    transactionTemplate.execute(status -> {
		        try {
		            // 业务操作1
		            // 业务操作2
		            return true; // 成功提交
		        } catch (Exception e) {
		            status.setRollbackOnly(); // 标记回滚
		            return false;
		        }
		    });
		}
		```
	2. XML配置事务：集中配置，与代码完全解耦。
		```
		<!-- 配置事务管理器 -->
		<bean id="transactionManager" 
		      class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
		    <property name="dataSource" ref="dataSource"/>
		</bean>
		
		<!-- 配置事务通知 -->
		<tx:advice id="txAdvice" transaction-manager="transactionManager">
		    <tx:attributes>
		        <tx:method name="save*" propagation="REQUIRED"/>
		        <tx:method name="get*" read-only="true"/>
		    </tx:attributes>
		</tx:advice>
		```
	3. 声明式事务(Spring的事务):通过AOP自动管理，代码简洁，业务逻辑与事务管理解耦。

	```
	@Service
	@Transactional // 类级别注解，所有公共方法都受事务管理
	public class UserService {
	    
	    @Transactional(propagation = Propagation.REQUIRES_NEW)
	    public void updateUser() {
	        // 方法自动在事务中执行
	    }
	}
	```

- Spring事务的属性设置
	 - [[事务传播]]行为
	 - 事务隔离级别
		 ```
		@Transactional(isolation = Isolation.READ_COMMITTED)
		public void transferMoney(Account from, Account to, BigDecimal amount) {
		    // 需要读取已提交的数据，避免脏读
		}
		```
	- 事务超时自动回滚设置:
		- `@Transactional(timeout = 30) // 30秒超时`
	- 只读事务优化:
		- `@Transactional(readOnly = true, timeout = 10)`
	- 异常回滚控制
		- 默认情况下:
			- Spring 事务只在遇到**运行时异常**（`RuntimeException`及其子类）和 **Error**​ 时才会回滚。
			- **受检异常**（Checked Exception）不会触发回滚。
		- 可以通过 `@Transactional`注解的 `rollbackFor`/ `rollbackForClassName`属性来自定义需要回滚的异常类型
			```
			@Transactional(
				rollbackFor = {BusinessException.class, SQLException.class}, // 这些异常回滚
				noRollbackFor = {IllegalArgumentException.class} // 这些异常不回滚
			)		
			```
- Spring[[事务失效]]的情况
- 事务的原理(AOP)
	- 事务的创建与开启
		1. **代理拦截**：Spring 容器会为标注了 `@Transactional`的类创建代理对象（JDK 动态代理或 CGLIB 代理）。当你调用一个事务方法时，实际上是在调用这个**代理对象**的方法。
		2. **执行拦截器**：代理对象会调用 `TransactionInterceptor`这个核心拦截器。拦截器的 `invoke(...)`方法中，会调用 `invokeWithinTransaction(...)`方法来处理事务逻辑。
		3. **获取事务**：在 `invokeWithinTransaction`方法中，Spring 会通过 `PlatformTransactionManager`事务管理器来**获取事务**。
			- 关键点在于 `getTransaction()`方法，它根据当前是否存在事务以及 `@Transactional`中定义的**传播行为**来决定是创建一个新事务，还是加入已有事务。
		4. **开启物理事务**：如果决定**开启新事务**，事务管理器会通过 `doBegin()`方法进行实际操作。对于 JDBC，其最核心的一步是从数据源获取连接，并调用 `connection.setAutoCommit(false)`来关闭自动提交，这标志着数据库事务的真正开始。

	- 提交与回滚的决策
		- **提交事务**：当你的业务方法**成功执行完毕**（未抛出异常），拦截器会调用事务管理器的 `commit()`方法，最终通过数据库连接的 `commit()`提交事务。
		    
		- **回滚事务**：
			- 如果业务方法执行过程中**抛出了异常**，拦截器会捕获到这个异常，并调用 `completeTransactionAfterThrowing(...)`方法。该方法会判断抛出的异常是否满足 `@Transactional`中配置的回滚规则（例如，默认情况下遇到运行时异常 `RuntimeException`和错误 `Error`会回滚）。如果满足条件，则通过数据库连接的 `rollback()`方法回滚事务。
		    

