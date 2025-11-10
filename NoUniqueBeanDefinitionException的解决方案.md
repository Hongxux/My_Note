---
aliases:
  - NoUniqueBeanDefinitionException
---
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