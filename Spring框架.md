## Spring框架的设计思想（引入IOC）

1. **目标**：我们希望软件设计符合 **[[OCP]]（开闭原则）**，即扩展新功能时无需修改已有代码。
    
2. **障碍**：传统的“上层依赖下层”的架构**违背了 DIP（[[依赖倒置原则]]）**，导致无法实现OCP。一旦下层变更，上层必然受牵连，需要修改。
	- **传统架构的问题**：
		- **常见情况**：`UserAction (表示层)`-> 依赖 `UserServiceImpl (业务层)`-> 依赖 `UserDaoImplForMySQL (持久层)`。
		- **违背 DIP**：这是一种“上层依赖下层”的紧耦合关系。**DIP 要求上层模块不应该依赖下层模块，而应共同依赖于抽象（接口）**。
		- **后果**：一旦想把数据库从 MySQL 切换到 Oracle，就需要修改 `UserServiceImpl`中 `new UserDaoImplForMySQL()`的代码——这**直接违背了 OCP**。
3. **解决方案的思想**：如何从“上层依赖下层”变成“上下层都依赖抽象”呢？答案是交出控制权。引入 **[[IOC]]（控制反转）** 思想，将对象的创建与依赖关系的控制权从程序内部转移到外部。
    - 控制反转的**反转的是什么**：
        - 将 **new 对象的控制权**交出去。
        - 将**维护对象间依赖关系的控制权**交出去。
    - **结果**：你的代码中不再出现 `new UserDaoImplForOracle()`这样的具体实现类，而是依赖于 `UserDao`接口。至于接口的具体实现是谁，程序不再关心。

4. **解决方案的实现**（图4）：**Spring 框架**作为 IoC 思想的容器，通过 **[[依赖注入]]（依赖注入）** 技术，成为了实现上述原则的标准方案。
	-  **依赖注入**：**是 IoC 思想最主流的实现方式，是“发动机”**。
	    - **如何实现**：Spring 容器通过 **setter 注入**或**构造器注入**的方式，将实现类（如 `UserDaoImplForOracle`）的实例“注入”到需要依赖的地方（如 `UserServiceImpl`的 `userDao`属性中）。
	    - **关系**：**IoC 是目标，DI 是手段**。**而 Bean 则是被 IOC 容器所管理的对象实例。**
	![[Pasted image 20251103190044.png]]
### 总结：一次完整的演进
Spring 框架就是一个实现了 IoC 思想的容器。

- **它的工作**：
    
    1. **创建对象**：根据配置（XML 或注解），帮你 `new`出所有需要管理的对象（Bean）。
        
    2. **维护关系**：根据依赖关系，通过 DI 将 Bean 组装在一起。
        
    3. **提供服务**：当你需要某个对象时，直接从它那里“获取”（`getBean`）。



假设我们要将数据访问层从 MySQL 切换到 Oracle：

2. **传统编码（违背OCP和DIP）**：
    
    ```
    public class UserServiceImpl {
        // 直接依赖具体实现类，是“上层依赖下层”
        private UserDao userDao = new UserDaoImplForMySQL();
        // 切换数据库？必须修改这里的代码！
    }
    ```
    
3. **面向接口编程（符合DIP，但未完全实现IoC）**：
    
    ```
    public class UserServiceImpl {
        private UserDao userDao; // 依赖接口
        // 但谁来做这个赋值？还是在代码某处需要 new
    }
    ```
    
4. **Spring IoC/DI（最终符合OCP和DIP）**：
    
    ```
    @Service
    public class UserServiceImpl {
        @Autowired // 由Spring容器注入，代码本身不创建对象
        private UserDao userDao; // 始终只依赖接口
    }
    ```
    
    **配置变更**：只需通过配置（如注解或XML）将 `UserDao`接口的实现类从 `UserDaoImplForMySQL`改为 `UserDaoImplForOracle`。**`UserServiceImpl`的代码一行都不用改！**

 
## AOP

## 事务
