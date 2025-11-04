---
aliases:
  - 依赖注入
---

### 一、 DI 核心定义与定位

#### 1. 一句话总结

​**DI 是一种通过外部容器在运行期自动将依赖对象注入到目标组件中，从而实现组件间解耦的特定设计模式。​**​

> ​**记忆钩子**​：当你的一个类（如 `ServiceA`）需要用到另一个类（如 `RepositoryB`）的功能时，不要自己 `new`，而是声明需要它，让框架（如 Spring）在运行时自动“注入”给你。

#### 2. 精确定义

- ​**定义**​：依赖注入是一种实现 ​**控制反转（IoC）​**​ 的设计模式。它指在运行期间，由框架（DI 容器）通过构造器、Setter 方法或字段反射等方式，将某个对象所依赖的其他对象（即其依赖项）动态地“注入”到其中。组件本身不负责创建或查找其依赖，而是被动接收依赖，从而实现了解耦。
    

#### 3. 关系辨析

- ​**DI 与 IoC**​：
    
    - ​**关系**​：​**DI 是 IoC 这种设计思想最主流、最具体的实现技术**。IoC 是“目标”（控制权反转），DI 是“手段”（通过注入实现反转）。
        
    - ​**辨析**​：IoC 是一个更宽泛的原则，除了 DI，理论上“服务定位器（Service Locator）”模式也能实现控制反转。但在 Spring 等现代框架中，当我们说 IoC 时，其实现机制默认就是指 DI。
        
    
- ​**DI 与 DL（依赖查找）​**​：
    
    - ​**关系**​：DL 是 DI 的**一种替代或前身**，是实现 IoC 的另一种方式，但现在已不常用。
        
    - ​**辨析**​：DL 是组件**主动**向容器“查找”依赖（如 JNDI lookup），是“拉（Pull）”的模式；而 DI 是容器**主动**将依赖“推送”给组件，是“推（Push）”的模式。​**DI 是更彻底的控制反转**。
        
    
- ​**DI 容器与工厂模式**​：
    
    - ​**关系**​：DI 容器是一个**更强大、更智能的“超级工厂”​**。
        
    - ​**辨析**​：简单工厂模式也负责创建对象，但 DI 容器在此基础上，还自动管理了对象的完整生命周期、依赖图谱的自动组装、配置集成等，其智能化和自动化程度远非普通工厂可比。
        
    

#### 4. 定位

- ​**所属范畴**​：DI 属于**软件工程**领域中的**设计模式/架构模式**范畴，是实现**依赖倒置原则（DIP）​**​ 这一面向对象设计原则的关键技术。
    
- ​**技术基础**​：其实现建立在 ​**Java 反射机制**​ 和 ​**动态代理**​ 之上，容器通过反射来分析类的依赖关系并完成注入。
    

#### 5. 涉及的设计理念与权衡

- ​**设计理念**​：“好莱坞原则”（Don't call us, we'll call you）和“关注点分离”。组件只关注自身业务逻辑，而依赖的创建、查找和组装等横切关注点交给容器处理。
    
- ​**优点**​：
    
    - ​**降低耦合度**​：组件不依赖于具体实现，而是依赖于抽象（接口），使得组件更独立、更易于替换和测试。
        
    - ​**提高可测试性**​：可以轻松注入 Mock 对象，实现独立的单元测试。
        
    - ​**增强可维护性**​：依赖关系清晰，代码更简洁，变更影响范围小。
        
    - ​**管理通用资源**​：容器可以统一管理如数据库连接、事务管理等通用依赖的生命周期。
        
    
- ​**缺点（权衡）​**​：
    
    - ​**框架依赖与复杂度**​：引入了特定的框架和容器概念，增加了系统的复杂性和学习曲线。
        
    - ​**调试难度**​：依赖关系由容器在运行时动态组装，当出现配置错误时，调用栈可能不直观，增加了调试难度。
        
    - ​**性能开销**​：基于反射的注入比直接 `new`有微小性能损耗，且容器启动时需要初始化整个对象图。
        
    
- ​**权衡结果**​：用**可控的框架复杂性和微小的性能开销**，换取整个软件生命周期内**巨大的可维护性、可测试性和灵活性提升**，这笔交易在绝大多数业务系统中是绝对值得的。
    

---

### 二、 经典使用场景

1. ​**场景描述：基于接口的 Service 层与 Repository 层解耦**​
    
    - ​**触发条件**​：在分层架构中，业务逻辑层（Service）需要调用数据访问层（Repository）的方法，但不希望直接依赖具体的数据访问实现（如 MySQL、Oracle）。
        
    - ​**关键特征**​：
        
        - `OrderService`依赖于 `OrderRepository`接口。
            
        - DI 容器在运行时将具体的 `JdbcOrderRepository`或 `JpaOrderRepository`实现注入到 `OrderService`中。
            
        - ​**价值**​：更换数据库访问技术时，只需更换注入的 Bean 实现，`OrderService`的代码无需任何改动。
            
        
    
2. ​**场景描述：单元测试中的 Mock 注入**​
    
    - ​**触发条件**​：需要对 `PaymentService`进行单元测试，但其依赖的 `EmailService`不稳定或不应在测试中触发真实邮件发送。
        
    - ​**关键特征**​：
        
        - 在测试代码中，配置 DI 容器（或使用 Mock 框架），使其向 `PaymentService`注入一个模拟的 `MockEmailService`。
            
        - ​**价值**​：实现了测试的隔离，可以专注测试 `PaymentService`的业务逻辑，测试速度快且结果稳定。
            
        
    

---

### 三、 工作原理与具体实现

DI 的核心流程是容器如何解析依赖关系并完成注入的。下图清晰地展示了这一过程：

```
flowchart TD
    A[“启动DI容器<br>（如ApplicationContext）”] --> B[“扫描与解析<br>获取BeanDefinition列表”]
    B --> C[“创建Bean实例<br>（反射调用构造器）”]
    C --> D[“解析依赖关系<br>（分析@Autowired等注解）”]
    D --> E[“递归注入依赖<br>（从容器中获取或创建依赖Bean）”]
    E --> F[“完成注入<br>（返回组装好的完整Bean）”]
    F --> G[“Bean就绪，可供使用”]

    D --> H[“是否存在循环依赖？”]
    H -- 是 --> I[“尝试解决<br>（如三级缓存）”]
    I --> E
    H -- 否 --> E
```

​**具体实现方式（三种主要注入方式）​**​：

1. ​**构造器注入**​：​**当前官方推荐的最佳实践**。
    
    ```
    @Service
    public class OrderService {
        private final OrderRepository repository;
        // 容器通过构造器注入依赖
        @Autowired // Spring 4.3 后，如果只有一个构造器可省略
        public OrderService(OrderRepository repository) {
            this.repository = repository;
        }
    }
    ```
    
    - ​**原理**​：容器调用目标类的构造方法，并将所需的依赖作为参数传入。
        
    - ​**优点**​：保证依赖不可变，对象在构造完成后即处于完全初始化的状态；能强制暴露循环依赖问题。
        
    - ​**潜在问题**​：当依赖过多时，构造函数参数列表会很长。
        
    
2. ​**Setter 方法注入**​：
    
    ```
    @Service
    public class OrderService {
        private OrderRepository repository;
        // 容器通过调用setter方法注入依赖
        @Autowired
        public void setRepository(OrderRepository repository) {
            this.repository = repository;
        }
    }
    ```
    
    - ​**原理**​：容器在调用无参构造器实例化对象后，再调用其 Setter 方法来设置依赖。
        
    - ​**优点**​：灵活性高，可以在运行时重新注入依赖。
        
    - ​**潜在问题**​：对象状态在生命周期内可能被修改，不如构造器注入安全。
        
    
3. ​**字段注入**​：​**不推荐用于主要业务逻辑**。
    
    ```
    @Service
    public class OrderService {
        @Autowired // 直接注入字段
        private OrderRepository repository;
    }
    ```
    
    - ​**原理**​：容器通过反射直接设置字段的值。
        
    - ​**缺点**​：破坏了封装性；使类难以脱离容器进行单元测试。
        
    

​**易出错点与解决措施**​：

- ​**问题1：循环依赖**。
    
    - ​**描述**​：Bean A 依赖 Bean B，同时 Bean B 又依赖 Bean A。
        
    - ​**触发条件**​：主要发生在构造器注入场景，或复杂的单例 Bean 的 Setter/字段注入场景。
        
    - ​**解决措施**​：
        
        - ​**预防**​：​**优先使用构造器注入**，它能在启动时立即暴露循环依赖，促使你改进设计（如提取公共功能到第三个 Bean 中）。
            
        - ​**框架解决**​：Spring 使用**三级缓存**机制解决单例 Bean 的 Setter/字段注入的循环依赖，但这是一种补救措施，应优先从设计上避免。
            
        
    
- ​**问题2：存在多个同类型候选 Bean（NoUniqueBeanDefinitionException）​**。
    
    - ​**描述**​：`@Autowired`注解默认是按类型（byType）进行自动装配的。​当有多个 Bean 都实现了同一个接口时，容器不知道注入哪一个。
        
    - ​**解决措施**​：
#### 方案一：使用 `@Primary`（设置主要候选者）

​**核心思想**​：当有多个同类型 Bean 时，将其中一个标记为“默认选项”。

​**适用场景**​：当多个实现中存在一个**最常用、最通用**的实现时。

​**代码示例**​：

```
// 将 EmpServiceA 设置为主要实现
@Service
@Primary // 关键注解：当有多个EmpService时，优先注入我
public class EmpServiceA implements EmpService {
    // ... 实现代码
}

@Service
public class EmpServiceB implements EmpService { // 这个是备选
    // ... 实现代码
}

@RestController
public class EmpController {
    @Autowired
    private EmpService emService; // 容器会优先注入EmpServiceA
}
```

​**优点**​：配置简单，只需在其中一个实现类上添加注解，所有按类型注入的地方都会自动生效。

​**缺点**​：不够灵活，如果不同地方需要注入不同的实现，则无法满足。

#### 方案二：使用 `@Qualifier`（按名称精确指定）

​**核心思想**​：不使用默认的类型匹配，而是**显式指定要注入的 Bean 的名称**。

​**适用场景**​：不同的注入点需要不同的实现。

​**代码示例**​：

```
@Service("empServiceA") // 为Bean指定一个名称，默认为类名首字母小写
public class EmpServiceA implements EmpService {
    // ... 实现代码
}

@Service("empServiceB")
public class EmpServiceB implements EmpService {
    // ... 实现代码
}

@RestController
public class EmpController {
    @Autowired
    @Qualifier("empServiceB") // 关键注解：显式指定要注入empServiceB
    private EmpService emService; // 容器会精确注入EmpServiceB
}
```

​**优点**​：非常灵活，可以针对每个注入点进行精确控制。

​**缺点**​：需要在每个注入点都指定名称，代码稍显繁琐。

#### 方案三：使用 `@Resource`（JSR-250 标准注解）

​**核心思想**​：使用 Java 标准注解 `@Resource`，它**默认按名称（byName）进行装配**。

​**适用场景**​：希望代码与 Spring 框架解耦，遵循 JSR-250 标准。

​**代码示例**​：

```
@Service("empServiceB") // 确保Bean有名称
public class EmpServiceB implements EmpService {
    // ... 实现代码
}

@RestController
public class EmpController {
    @Resource(name = "empServiceB") // 按名称注入。如果name省略，则使用字段名(emService)去找同名的Bean
    private EmpService emService;
}
```

​**`@Resource`与 `@Autowired`的区别**​：

- `@Autowired`是 Spring 框架的，默认 byType。
    
- `@Resource`是 Java 标准（JSR-250），默认 byName。它的行为更可预测。
    

---

### 四、 面试官关心的问题与答案

1. ​**Q：什么是依赖注入（DI）？它解决了什么问题？​**​
    
    - ​**A**​：DI 是一种设计模式，由框架在运行时将组件所依赖的其他对象注入进去，而非组件自己创建或查找依赖。它**解决了组件间紧耦合的问题**，使得代码更易测试、维护和扩展，是实现控制反转（IoC）思想的主要手段。
        
    
2. ​**Q：依赖注入有几种方式？你推荐哪种？为什么？​**​
    
    - ​**A**​：主要有三种：构造器注入、Setter 注入和字段注入。我**强烈推荐使用构造器注入**。因为它能保证依赖不可变和完全初始化，避免了空指针异常，并且能强制暴露循环依赖问题，使代码更健壮、更安全，也更易于测试。
        
    
3. ​**Q：Spring 是如何解决循环依赖的？​**​
    
    - ​**A**​：Spring 通过**​“三级缓存”​**​ 机制来解决**单例 Bean 的 Setter/字段注入**​ 的循环依赖。三级缓存分别存储：1）完整的单例对象；2）早期的半成品对象（已实例化但未注入依赖）；3）Bean 工厂。当发生循环依赖时，容器会通过提前暴露一个早期引用（存放在二级缓存）来打破循环。但**构造器注入的循环依赖无法解决**。
        
    
4. ​**Q：`@Autowired`和 `@Resource`有什么区别？​**​
    
    - ​**A**​：
        
        - `@Autowired`是 Spring 框架的注解，默认按**类型（byType）​**​ 进行自动装配。配合 `@Qualifier`可指定名称。
            
        - `@Resource`是 JSR-250 标准注解，默认按**名称（byName）​**​ 进行装配。如果未指定名称，则回退到按类型装配。
            
        - 推荐在纯 Spring 环境使用 `@Autowired`，若希望代码与 Spring 解耦，可使用 `@Resource`。
            
        
    
5. ​**Q：为什么说依赖注入有利于单元测试？​**​
    
    - ​**A**​：因为被测试的组件其依赖是通过接口注入的，而非在内部硬编码。在测试时，我们可以轻松地使用 Mock 框架（如 Mockito）创建一个依赖的模拟对象（Mock），并将其注入到被测试组件中。这样可以**隔离被测试代码**，仅验证其业务逻辑，使测试更专注、快速和稳定。
        
    

希望这份严谨、系统的梳理能帮助你彻底理解依赖注入（DI）这一至关重要的设计模式。