### [[Spring事务原理]]
### Spring事务的演变
 #### 1. **编程式事务（手动管理）**

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

**特点**：灵活但繁琐，需要在代码中显式控制事务边界。
#### 2. **XML配置事务**

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

**特点**：集中配置，与代码完全解耦。
#### 3.声明式事务(Spring的事务)

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

**特点**：通过AOP自动管理，代码简洁，业务逻辑与事务管理解耦。

---

### Spring事务的属性设置
#### 基本属性
##### 一、[[事务传播]]行为（Propagation）- 核心难点

##### 二、事务隔离级别（Isolation）

解决并发事务间的数据可见性问题。

|隔离级别|脏读|不可重复读|幻读|性能|说明|
|---|---|---|---|---|---|
|**READ_UNCOMMITTED**|❌|❌|❌|最高|可读取未提交数据|
|**READ_COMMITTED**|✅|❌|❌|较高|Oracle默认，解决脏读|
|**REPEATABLE_READ**|✅|✅|❌|中等|MySQL默认，解决不可重复读|
|**SERIALIZABLE**|✅|✅|✅|最低|串行化，解决所有问题|

```
@Transactional(isolation = Isolation.READ_COMMITTED)
public void transferMoney(Account from, Account to, BigDecimal amount) {
    // 需要读取已提交的数据，避免脏读
}
```

#### 高级属性

##### 1. 事务超时设置

```
@Transactional(timeout = 30) // 30秒超时
public void batchProcess() {
    // 长时间批处理操作
    // 如果超过30秒，自动回滚
}
```

##### 2. 只读事务优化

```
@Transactional(readOnly = true, timeout = 10)
public List<User> searchUsers(String keyword) {
    // 纯查询操作，启用只读优化
    return userDao.findByKeyword(keyword);
}
```

##### 3. 异常回滚控制
默认情况下，Spring 事务只在遇到**运行时异常**（`RuntimeException`及其子类）和 **Error**​ 时才会回滚。**受检异常**（Checked Exception）不会触发回滚。

你可以通过 `@Transactional`注解的 `rollbackFor`/ `rollbackForClassName`属性来自定义需要回滚的异常类型。
```
@Transactional(
    rollbackFor = {BusinessException.class, SQLException.class}, // 这些异常回滚
    noRollbackFor = {IllegalArgumentException.class} // 这些异常不回滚
)
public void businessOperation() throws BusinessException {
    // 业务逻辑
    if (invalidCondition) {
        throw new IllegalArgumentException("参数错误"); // 不回滚
    }
    if (businessFailed) {
        throw new BusinessException("业务失败"); // 回滚
    }
}
```

### Spring[[事务失效]]的情况
### 事务管理器（PlatformTransactionManager）

Spring事务抽象的核心接口，支持不同的事务API。

```
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

**常见实现**：

- `DataSourceTransactionManager`：JDBC事务
    
- `JpaTransactionManager`：JPA事务
    
- `JtaTransactionManager`：分布式事务
    