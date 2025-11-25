---
aliases:
  - "@ConfigurationProperties"
---
好的，这六张图片共同系统地阐述了在 Spring Boot 应用中，如何优雅地实现 **“基于外部配置的属性注入”** 来驱动业务 Bean，其核心设计思想是**解耦**与**外部化配置**。

我自己有一套默认值，而你也可以在配置文件中设置自己想要的值来覆盖我的值

---

### 一、 核心架构：三部分解耦

这套方案将应用分为三个清晰且解耦的部分：

1. **属性封装类（`CartoonProperties`）**：负责**定义配置元数据**并与配置文件绑定。
    
2. **配置文件（`application.yml`）**：负责**提供具体的属性值**，实现配置外部化。
    
3. **业务功能Bean（`CartoonCatAndMouse`）**：包含核心逻辑，**通过依赖注入**获取配置属性。
    

---

### 二、 实现步骤详解

#### 1. 创建属性封装类
[[@ConfigurationProperties]]


#### 2. 在配置文件中提供属性值

然后在 `application.yml`中，按照约定的结构提供具体值。

```
# 步骤2：在外部配置文件中设置属性值
cartoon: # 对应prefix
  cat:
    name: "图多盖洛" # 对应CartoonProperties.cat.name
    age: 5 # 对应CartoonProperties.cat.age
  mouse:
    name: "泰菲"
    age: 1
```

这种方式**将配置与代码分离**。要修改猫和老鼠的名字或年龄，你**无需重新编译Java代码**，只需修改这个配置文件即可。

#### 3. 创建业务功能Bean并注入属性

接下来，创建业务逻辑类，并让它使用上面定义的属性。

**不推荐的方式**：直接使用 `@Import`强制加载。

```
@Component
public class CartoonCatAndMouse {
    private Cat cat;
    private Mouse mouse;
    // ... 直接new对象，紧耦合，不灵活
}
```

**推荐的方式**：通过构造器注入 `CartoonProperties`对象。

```
// 步骤3：定义业务Bean，并激活属性绑定
@Component
@EnableConfigurationProperties(CartoonProperties.class) // 关键注解：激活属性绑定并将CartoonProperties变为Bean
public class CartoonCatAndMouse {
    private CartoonProperties cartoonProperties; // 依赖属性对象

    // 通过构造器注入（推荐）
    public CartoonCatAndMouse(CartoonProperties cartoonProperties) {
        this.cartoonProperties = cartoonProperties;
        // 设定默认值，如果配置文件中有则覆盖
        
        // ... 初始化mouse
    }

    public void play() {
        System.out.println(cartoonProperties.getCat().getName() + "和" + cartoonProperties.getMouse().getName() + "打起来了");
    }
}
```
![[Pasted image 20251113185831.png]]
- **`@EnableConfigurationProperties(CartoonProperties.class)`**：这个注解有两个重要作用：
    
    1. **激活绑定**：确保被 `@ConfigurationProperties`标注的 `CartoonProperties`类被Spring处理，并将配置文件中的值注入进去。
        
    2. **注册Bean**：将 `CartoonProperties`类本身也注册为一个Spring容器的Bean，这样它才能被注入到 `CartoonCatAndMouse`中。
        
    

#### 4. 最终效果

应用启动后，Spring会自动完成以下步骤（对应图1）：

1. 扫描到 `@EnableConfigurationProperties`，从而将 `CartoonProperties`注册为Bean，并完成属性注入。
    
2. 因为 `CartoonCatAndMouse`上有 `@Component`，Spring要创建它的实例。
    
3. 发现其构造器需要 `CartoonProperties`类型的参数，于是将从容器中已创建好的 `CartoonProperties`Bean注入进来。
    
4. 业务Bean `CartoonCatAndMouse`成功创建，并可以正常使用配置属性。
    

---

### 三、 设计精髓与总结（图6）

图6的小结精准地概括了这种方案的价值和设计哲学：

1. **业务bean的属性可以为其设定默认值**：你可以在 `CartoonProperties`类中给字段赋默认值。如果配置文件中没有设置，则使用默认值，保证了健壮性。
    
2. **当需要设置时通过配置文件传递属性**：实现了**配置外部化**，这是云原生和12-Factor应用的核心要求，使应用更容易适配不同环境（开发、测试、生产）。
    
3. **业务bean应尽量避免设置强制加载**：使用 `@EnableConfigurationProperties`是一种**条件性**的加载。它只在真正需要用到这个业务Bean的时候，才会触发其依赖的属性的加载和Bean的创建。这相比使用 `@Import`或直接写死值的**强制加载**，**降低了Spring容器的管理负担**，提高了应用的启动速度和灵活性。
    

**总结**：

这六张图连起来，展示的是 Spring Boot **“约定大于配置”** 和 **“控制反转”**思想的经典实践。它通过：

- **`@ConfigurationProperties`**：实现了**类型安全**的配置映射。
    
- **`@EnableConfigurationProperties`**：实现了**按需加载**和**依赖注入**。
    

这套模式是开发高质量、可配置、易维护的 Spring Boot 应用的**标准做法**，被广泛应用于各种 Starter 和业务模块的开发中。