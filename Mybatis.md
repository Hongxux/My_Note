### quick start

![[Pasted image 20251105093145.png]]
![[Pasted image 20251105093204.png]]
![[Pasted image 20251105093322.png]]
![[Pasted image 20251105093333.png]]

### IDEA配置SQL提示
![[Pasted image 20251105094110.png]]

![[Pasted image 20251105094054.png]]
### 数据库连接池
![[Pasted image 20251105094717.png]]



#### 一、 数据库连接池是什么？

根据第一张图，​**数据库连接池**是一个负责分配和管理数据库连接的容器。它的核心作用包括：

- ​**连接管理**​：应用程序需要数据库连接时，从池中获取一个现有的连接，而不是每次重新创建新连接。
    
- ​**连接复用**​：使用完毕后，连接被归还到池中，供其他请求重复使用，避免频繁创建和销毁连接的开销。
    
- ​**资源回收**​：自动释放空闲时间过长的连接，防止因连接未正确关闭而导致的数据库连接泄漏（连接遗漏）。
    

​**简单来说**​：连接池就像是一个“共享单车站”，应用程序需要时直接取用一辆空闲单车（连接），用完后归还，由站点统一维护，避免了每次都要买新车（创建新连接）的浪费。

---

#### 二、 为什么使用数据库连接池？其优势是什么？

1. ​**资源重用**​：大幅减少数据库连接创建和销毁的次数，降低系统资源消耗。
    
2. ​**提升系统响应速度**​：应用程序无需等待建立新的数据库连接（这是一个耗时的网络和认证过程），直接从池中获取已建立的连接，响应速度更快。
    
3. ​**避免数据库连接遗漏**​：通过统一的连接生命周期管理，确保连接在使用后能被正确回收，防止连接累积耗尽数据库资源，导致系统崩溃。
    

---

#### 三、 技术标准与主流产品

- ​**标准接口**​：`javax.sql.DataSource`
    
    - 这是由 Sun（现 Oracle）官方提供的接口，定义了获取连接的方法 `Connection getConnection()`。
        
    - 各家第三方组织（如阿里巴巴、Apache等）实现此接口，提供了各自的连接池产品。
        
    
- ​**主流产品对比**​：
    
    - ​**C3P0**​：较早期的开源连接池，稳定但性能较新方案有差距。
        
    - ​**DBCP**​：Apache Commons 项目下的连接池。
        
    - ​**Druid**​：阿里巴巴开源项目，功能全面，提供强大的监控和统计功能，在国内非常流行。
        
    - ​**HikariCP**​：以**高性能**著称，是 ​**Spring Boot 2.x 及以上版本的默认连接池**，追求极致速度。
        
    

---

#### 四、 实战配置：在 Spring Boot 中集成 Druid 连接池

第四张图提供了一个具体的示例，展示了如何将 Spring Boot 项目的默认连接池切换为功能强大的 ​**Druid**。

​**配置步骤如下：​**​

1. ​**添加 Maven 依赖**​（在 `pom.xml`中）：
    
    ```
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.2.8</version> <!-- 建议使用最新稳定版本 -->
    </dependency>
    ```
    
2. ​**修改配置文件**​（在 `application.properties`或 `application.yml`中）：
    
    ```
    # 配置数据源类型为 Druid
    spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
    # 数据库连接信息
    spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
    spring.datasource.url=jdbc:mysql://localhost:3306/mybatis
    spring.datasource.username=root
    spring.datasource.password=1234
    ```
    
3. ​**​（可选）开启监控功能**​：Druid 的一大优势是内置了监控平台，你可以在配置中启用它，通过浏览器访问来查看详细的SQL监控、统计信息，便于性能分析和故障排查。
    
### [[lombok]]
