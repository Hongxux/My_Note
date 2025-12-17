---
aliases:
  - Mybatis Plus
  - SSMP
---



---

### 一、 基础整合：配置与设置

图1和图2展示了整合 MyBatis-Plus 的前两步：**引入依赖**和**基础配置**。

#### 1. 导入 Starter 依赖

MyBatis-Plus 提供了专门的 Spring Boot Starter，极大简化了依赖管理。
**SpringBoot2**
```
<!--SpringBoot2->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.4.3</version> <!-- 请使用最新版本 -->
</dependency>
<!-- 配套的Druid数据源 Starter -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid-spring-boot-starter</artifactId>
    <version>1.2.6</version>
</dependency>
```
---
**SpringBoot3**
- MyBatis Plus拆分成了主要组件和一些插件（例如分页），于是引入版本管理
	```
	<dependencyManagement>  
		<dependencies>
			<dependency>
					<groupId>com.baomidou</groupId>  
					<artifactId>mybatis-plus-bom</artifactId>  
					<version>3.5.14</version>  
					<type>pom</type>  
					<scope>import</scope>  
			</dependency>
		</dependencies>
	</dependencyManagement>
	```
- 主要组件
```
<dependency>  
	<groupId>com.baomidou</groupId>  
	<artifactId>mybatis-plus-spring-boot3-starter</artifactId>  
</dependency>
```
- 分页所需插件
```
<dependency>  
	<groupId>com.baomidou</groupId>  
	<artifactId>mybatis-plus-jsqlparser-4.9</artifactId>  
</dependency>
```
**说明**：

- `mybatis-plus-boot-starter`会自动引入 MyBatis、MyBatis-Plus 核心库以及 Spring Boot 相关的适配库，**无需再单独引入 MyBatis 的依赖**。
    
- 使用 `druid-spring-boot-starter`可以方便地集成阿里巴巴的 Druid 连接池，这是一个用于监控和管理数据库连接的高性能组件。
    

#### 2. 基础配置

在 `application.yml`中配置数据源和 MyBatis-Plus 的全局设置。

```
spring:
  datasource:
    druid:
      driver-class-name: com.mysql.cj.jdbc.Driver # MySQL驱动
      url: jdbc:mysql://localhost:3306/ssm_db?serverTimezone=UTC
      username: root
      password: root

mybatis-plus:
  global-config:
    db-config:
      table-prefix: tbl_  # 全局表前缀配置
      id-type: auto       # ID生成策略，AUTO代表数据库自增
```

**配置解析**：

- `spring.datasource.druid.*`：配置**数据源**，这是连接数据库的基础信息。
    
- `mybatis-plus.global-config.db-config.table-prefix`：**全局表前缀**。如果所有表都有共同的前缀（如 `tbl_user`），在此配置后，在代码中只需写实体名（`User`），MP 会自动将表名拼接为 `tbl_user`。
    
- `mybatis-plus.global-config.db-config.id-type`：**主键生成策略**。
    
    - `auto`：数据库自增（适用于 MySQL 等）。
        
    - `assign_id`或 `assign_uuid`：MP 使用雪花算法生成一个 Long 型或 String 型的 ID（分布式场景常用）。
        
#### 3.开启[[MP运行日志]]

---
## 数据层
### 一、 核心用法：通用 CRUD
#### 1. 创建 Dao 接口

```
import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;

@Mapper // 关键注解，声明这是一个MyBatis的Mapper接口，让Spring能扫描到它
public interface BookDao extends BaseMapper<Book> {
    // 无需编写任何方法！
}
```

**关键点**：

- **继承 `BaseMapper<T>`**：其中泛型 `T`是你的**实体类**（如 `Book`）。这表示这个 Dao 接口用于操作 `Book`表。
    
- **`@Mapper`注解**：务必添加此注解，或将接口所在路径添加到 `@MapperScan`中，否则 Spring 无法发现并代理这个接口。
    

#### 2. 直接使用强大的 CRUD 方法

一旦你的 Dao 接口继承了 `BaseMapper`，你就可以直接使用它提供的数十个通用方法，**无需编写对应的 SQL 语句或 XML 文件**。

**在 Service 中注入并使用**：

```
@Service
public class BookServiceImpl implements BookService {

    @Autowired
    private BookDao bookDao;

    @Override
    public Book getById(Long id) {
        // 按ID查询: SELECT * FROM tbl_book WHERE id = ?
        return bookDao.selectById(id);
    }

    @Override
    public boolean save(Book book) {
        // 插入: INSERT INTO tbl_book (id, name, ...) VALUES (?, ?, ...)
        return bookDao.insert(book) > 0;
    }

    @Override
    public boolean update(Book book) {
        // 按ID更新: UPDATE tbl_book SET ... WHERE id = ?
        return bookDao.updateById(book) > 0;
    }

    @Override
    public boolean delete(Long id) {
        // 按ID删除: DELETE FROM tbl_book WHERE id = ?
        return bookDao.deleteById(id) > 0;
    }

    @Override
    public List<Book> getAll() {
        // 查询所有: SELECT * FROM tbl_book
        return bookDao.selectList(null); // null 代表没有查询条件
    }
}
```

**常用方法列表**（`BaseMapper`提供的一部分）：

|方法|SQL 等价操作|说明|
|---|---|---|
|`int insert(T entity)`|`INSERT INTO ...`|插入一条记录|
|`int deleteById(Serializable id)`|`DELETE FROM ... WHERE id = ?`|根据主键删除|
|`int updateById(T entity)`|`UPDATE ... WHERE id = ?`|根据主键更新|
|`T selectById(Serializable id)`|`SELECT * FROM ... WHERE id = ?`|根据主键查询|
|`List<T> selectList(Wrapper<T> queryWrapper)`|`SELECT * FROM ...`|查询列表（可带条件）|
|`List<T> selectBatchIds(Collection idList)`|`SELECT * FROM ... WHERE id IN (?,?)`|根据ID集合批量查询|

---

### 二、 高级用法：条件构造器与分页查询

除了通用 CRUD，MyBatis-Plus 还提供了两个极为强大的功能。

#### 1. 条件构造器 `QueryWrapper`
![[Pasted image 20251111153227.png]]
用于动态构建复杂的查询条件，避免手动拼接 SQL 字符串。

```
// 示例：查询名字包含"Spring"且价格低于50的书
QueryWrapper<Book> queryWrapper = new QueryWrapper<>();
queryWrapper.like("name", "Spring") // WHERE name LIKE '%Spring%'
            .lt("price", 50);       // AND price < 50

List<Book> books = bookDao.selectList(queryWrapper);
```

**常用条件方法**：

- `eq("column", value)`：`=`
    
- `ne("column", value)`：`!=`
    
- `gt("column", value)`：`>`
    
- `lt("column", value)`：`<`
    
- `like("column", value)`：`LIKE '%value%'`
    
- `orderByAsc("column")`：`ORDER BY column ASC`
    
##### **`LambdaQueryWrapper`**

```
@Test
void testGetByCondition(){
    LambdaQueryWrapper<Book> lqw = new LambdaQueryWrapper<Book>();
    lqw.like(Book::getName, "Spring"); // 使用Lambda表达式引用实体类属性
    bookDao.selectList(lqw);
}
```

- **优点**：**类型安全**。`Book::getName`通过方法引用来指定字段，编译器会检查该字段是否存在于 `Book`类中。如果字段名修改，IDE 会自动提示更新，**有效避免因字段名拼写错误导致的运行时 Bug**。
    
- **场景**：**绝大多数情况下，都应使用此方式**。
    
##### 动态查询（`condition`参数）
**错误代码**
```
@Test
void testGetBy2(){
    String name = null; // 查询条件为null
    LambdaQueryWrapper<Book> lqw = new LambdaQueryWrapper<Book>();
    lqw.like(Book::getName, name); // 将null值传入条件
    bookDao.selectList(lqw);
}
```

**运行结果**：生成的 SQL 为 `WHERE (name LIKE ?)`，参数为 `%null%`。这会导致查询不到任何数据（因为数据库中几乎没有字段值等于字符串 `"null"`的记录），这显然不是我们想要的。

**预期目标**：当 `name`参数为 `null`或空字符串时，应该**忽略**这个查询条件，查询所有数据。

###### 解决方案：使用 `condition`参数实现动态查询

MyBatis-Plus 的绝大多数条件方法都重载了一个带 `boolean condition`参数的版本。这是实现**动态条件拼装**的关键。

**正确写法**：

```
@Test
void testGetBy2(){
    String name = null; // 来自前端的参数，可能为null
    LambdaQueryWrapper<Book> lqw = new LambdaQueryWrapper<Book>();
    
    // 只有当name不为null且不为空字符串时，才添加like条件
    lqw.like(StringUtils.isNotBlank(name), Book::getName, name);
    
    bookDao.selectList(lqw);
}
```

**代码解读**：

- `like(boolean condition, ...)`：第一个参数是一个布尔值条件。
    
- `StringUtils.isNotBlank(name)`：这是一个判断条件，当 `name`不为 `null`、不是空字符串、不是纯空格时返回 `true`。
    
- **执行逻辑**：
    
    - 如果 `name`有效（如 `"Spring"`），条件为 `true`，则生成的 SQL 是：`WHERE (name LIKE '%Spring%')`。
        
    - 如果 `name`无效（为 `null`），条件为 `false`，则 **MyBatis-Plus 会跳过此条件**，生成的 SQL 是：`WHERE`子句为空，即查询所有数据。
        
    

**其他动态条件示例：**


```
LambdaQueryWrapper<Book> lqw = new LambdaQueryWrapper<>();
// 动态等值查询
lqw.eq(price != null, Book::getPrice, price);
// 动态范围查询
lqw.between(startDate != null && endDate != null, Book::getPublishDate, startDate, endDate);
// 动态排序
lqw.orderBy(asc != null, true, Book::getPrice);
```
#### 2. 分页查询 `selectPage()`
![[Pasted image 20251111152615.png]]
需要先配置分页插件，然后使用 `Page`对象进行分页查询。

**配置分页插件**（创建一个配置类）：固定写法，不要纠结
MyBatis-Plus 的分页功能**不是默认开启的**，需要手动配置一个**拦截器（Interceptor）**。这是因为分页的实现需要在 SQL 执行前，动态地拼接 `LIMIT`等分页语句。

```
import com.baomidou.mybatisplus.annotation.DbType;
import com.bamou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.inner.PaginationInnerInterceptor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加分页拦截器，指定数据库类型为 MYSQL
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```
- 于 `v3.5.9` 起，`PaginationInnerInterceptor` 已分离出来。如需使用，则需单独引入 `mybatis-plus-jsqlparser` 依赖 ， 具体请查看 [安装](https://baomidou.com/getting-started/install) 一章。
- 
**使用分页查询**：

```
// 查询第2页，每页5条数据
Page<Book> page = new Page<>(2, 5);
// 可以附加查询条件
QueryWrapper<Book> queryWrapper = new QueryWrapper<>();
queryWrapper.like("name", "Java");
bookDao.selectPage(page, queryWrapper);

```
执行查询后，所有的分页数据和结果都**被自动填充到了你传入的 `page`对象中**。

|方法|说明|对应 SQL|
|---|---|---|
|`page.getRecords()`|**当前页的数据列表**（`List<Book>`）|查询得到的结果集|
|`page.getCurrent()`|**当前页码**|你设置的 `current`|
|`page.getSize()`|**每页显示条数**|你设置的 `size`|
|`page.getTotal()`|**总记录数**（数据库中共有多少条符合条件的数据）|`COUNT(*)`|
|`page.getPages()`|**总页数**（由 `total/size`计算得出）|计算字段|
## Service层

^ed9492

**不要从零开始编写业务层代码，而是直接继承MyBatis-Plus提供的现成模板，然后按需进行定制化开发。**

### 一、 快速开发方案详解

#### 1. 使用MP提供的通用接口与实现类

MyBatis-Plus 不仅为数据层（Dao/Mapper）提供了 `BaseMapper`，也为业务层（Service）提供了通用的模板。

- **通用业务接口**：`IService<T>`
    
    - 这个接口定义了大量的**通用业务方法**，如 `save`（增）、`remove`（删）、`update`（改）、`get`（查）、`list`（批量查）、`page`（分页查）等。这些方法直接基于你继承的 `BaseMapper`操作。
        
    
- **通用业务实现类**：`ServiceImpl<M, T>`
    
    - 这个类实现了 `IService<T>`接口的所有通用方法。
        
    - 它需要两个泛型参数：
        
        - `M`： 你的 `Mapper`接口类型（继承自 `BaseMapper`）。
            
        - `T`： 你的实体类类型。
            
        
    

#### 2. 具体实现步骤

**第一步：让你的业务接口继承 `IService<T>`**

```
// 你的业务接口
public interface BookService extends IService<Book> { // 继承MP的通用业务接口
    // 在此处声明你自己的、非通用的业务方法
    boolean someCustomBusinessLogic(Long bookId);
}
```

**第二步：让你的业务实现类继承 `ServiceImpl<M, T>`并实现你的业务接口**

```
// 你的业务实现类
@Service // 声明为Spring的Service组件
public class BookServiceImpl
       extends ServiceImpl<BookDao, Book> // 关键：继承MP的通用实现类，并指定Mapper和Entity
       implements BookService { // 实现你自己的业务接口

    @Override
    public boolean someCustomBusinessLogic(Long bookId) {
        // 1. 你可以直接使用父类ServiceImpl中提供的强大方法
        Book book = this.getById(bookId); // this.getById() 方法来自ServiceImpl

        // 2. 编写你的自定义业务逻辑...
        if(book != null) {
            // ... 一些处理
            return this.updateById(book); // this.updateById() 方法也来自ServiceImpl
        }
        return false;
    }
}
```

**解释**：

- 通过继承 `ServiceImpl<BookDao, Book>`，你的 `BookServiceImpl`类**瞬间就获得了**几十个开箱即用的 CRUD 方法，如 `getById`, `save`, `update`, `remove`, `list`, `page`等。
    
- 在你的自定义方法中，可以通过 `this.`直接调用这些方法，**无需再注入 `BookDao`并编写基础代码**。
    

### 二、 功能重载与追加的原则

图中特别强调了一个重要原则：**“注意重载时不要覆盖原始操作，避免原始提供的功能丢失”**。

这指的是当你在自己的业务接口（如 `BookService`）中**“重载”**父接口（`IService`）的方法时，要非常小心。

#### 1. 功能重载（Overload）

这是指在同一个类中，方法名相同但参数列表不同。这是安全的。

```
public interface BookService extends IService<Book> {
    // 重载：方法名也是getById，但参数不同
    Book getById(String isbn); // 根据ISBN查询
    // 父接口的 getById(Serializable id) 方法依然存在
}
```

#### 2. 功能覆盖/重写（Override）与最佳实践

**需要警惕的是“覆盖”**。如果你在子接口中声明了一个与父接口**完全一样的方法签名**，那么它就会覆盖父接口的方法。

**不推荐的危险做法**：

```
public interface BookService extends IService<Book> {
    // 重写：与IService中的方法签名完全相同
    boolean save(Book entity);
}
```

这样做，会导致从 `IService`继承来的那个功能完善的 `save`方法被隐藏，你**必须**在 `BookServiceImpl`中重新实现它，否则会报错，这完全违背了“快速开发”的初衷。

**推荐的“功能追加”做法**：

更好的方式是为新功能起一个不同的名字，或者在原有功能基础上进行**增强**。

- **方式一：追加新方法**（最安全）
    
    ```
    public interface BookService extends IService<Book> {
        // 追加一个全新的方法，不干扰原有功能
        boolean saveWithDetailedLog(Book entity);
    }
    ```
    
- **方式二：在实现类中增强**（AOP思想）
    
    ```
    @Service
    public class BookServiceImpl extends ServiceImpl<BookDao, Book> implements BookService {
        // 不覆盖save方法，而是直接使用父类的save方法
    
        @Override
        public boolean someCustomBusinessLogic(Book book) {
            // 在调用父类save方法前后，可以添加自定义逻辑
            System.out.println("准备保存图书：" + book.getName());
            boolean result = super.save(book); // 调用父类（ServiceImpl）的save方法
            System.out.println("图书保存结果：" + result);
            return result;
        }
    }
    ```
    
