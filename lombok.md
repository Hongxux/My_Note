
![[Pasted image 20251105102619.png]]
Lombok 是一个旨在帮助 Java 开发者减少冗余代码、提升开发效率的类库。其核心思想是 ​**​“通过注解来简化代码”​**。

#### 一、 Lombok 是什么？

图1的开头给出了精准的定义：

> ​**Lombok 是一个实用的 Java 类库，能通过注解的形式自动生成构造器、getter/setter、equals、hashCode、toString 等方法，并可以自动化生成日志变量，简化 Java 开发、提高效率。​**​

- ​**目标**​：解决 Java 开发中大量重复、模板化的代码问题（例如为每个字段生成 getter 和 setter 方法）。
    
- ​**工作原理**​：在**编译阶段**，Lombok 会根据你在类上添加的注解，自动将对应的代码（如 getter、setter）注入到生成的 `.class`字节码文件中。​**你的源代码文件 `.java`仍然保持简洁**。
    
- ​**价值**​：让开发者更专注于业务逻辑，而非重复的代码编写。
    

#### 二、 Lombok 常用注解详解

图1中的表格是 Lombok 的精华，列出了最核心、最常用的注解：

|注解|作用|说明与示例|
|---|---|---|
|​**`@Getter`/ `@Setter`**​|为所有属性生成 getter 和 setter 方法。|无需再手动编写 `public String getName() { return name; }`这样的方法。|
|​**`@ToString`**​|生成包含所有类属性的 `toString()`方法。|打印对象时，输出格式如 `User(name=张三, age=20)`，便于调试。|
|​**`@EqualsAndHashCode`**​|基于所有非静态字段生成 `equals()`和 `hashCode()`方法。|用于对象比较和集合类（如 HashSet, HashMap）的正确操作。|
|​**`@NoArgsConstructor`**​|生成一个无参构造函数。|例如 `public User() {}`。|
|​**`@AllArgsConstructor`**​|生成一个包含所有成员变量的构造函数。|例如 `public User(String name, Integer age) { ... }`。|
|​**`@Data`**​|​**最常用的注解**，是以下注解的集合体：  <br>`@Getter`+ `@Setter`+ `@ToString`+ `@EqualsAndHashCode`+ `@RequiredArgsConstructor`。|一个 `@Data`注解就能解决绝大多数 POJO（实体）类的代码生成需求。|

​**示例对比：​**​

​**使用 Lombok 前：​**​

```
public class User {
    private Long id;
    private String name;
    private Integer age;

    // 必须手动编写大量的模板代码
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    // ... 为 name 和 age 生成 getter/setter
    // ... 生成 toString()
    // ... 生成 equals() 和 hashCode()
    // ... 生成构造方法
}
```

​**使用 Lombok 后：​**​

```
@Data // 一个注解搞定所有
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
    private Integer age;
    // 不需要再写任何getter, setter, toString等方法！
}
```

两个类编译后产生的字节码功能是完全等价的，但后者代码极其简洁。

#### 三、 如何在项目中使用 Lombok？

图2展示了如何通过 Maven 将 Lombok 集成到你的项目中。

1. ​**添加依赖**​：在项目的 `pom.xml`文件中的 `<dependencies>`部分添加以下依赖：
    
    ```
    <dependency>
        <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
        <version>1.18.30</version> <!-- 建议使用最新稳定版本 -->
        <scope>provided</scope>
    </dependency>
    ```
    
    - ​**`groupId`和 `artifactId`**​：这是 Lombok 在 Maven 中央仓库的唯一坐标。
        
    - ​**`scope`**​：设置为 `provided`，是因为 Lombok 仅在编译时起作用，最终打包时不需要将其打入部署包。
        
    
2. ​**IDE 支持**​：为了让你的 IDE（如 IntelliJ IDEA, Eclipse）能够识别 Lombok 注解并提供代码提示，​**必须安装对应的 Lombok 插件**。这是一个关键步骤，否则 IDE 会报错“找不到 getter/setter 方法”。
    