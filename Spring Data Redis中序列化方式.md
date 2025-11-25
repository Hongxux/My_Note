### 总结与建议

- **默认的JDK序列化方式必须被替换**。
    
- 对于**大多数生产环境**，推荐使用 **`StringRedisTemplate`+ 手动JSON序列化**的方案。它在内存效率、灵活性和跨语言兼容性上取得了最佳平衡。
    
- 虽然手动序列化会增加少量代码，但可以通过封装工具类来简化，其带来的性能优势和架构清晰度是值得的。
    
### 核心结论：两种主流的序列化方案对比

图片展示了Spring Data Redis中处理序列化的两种典型方案。它们的对比如下：

| 特性        | 方案一：自定义 `RedisTemplate`(使用JSON序列化)          | 方案二：使用 `StringRedisTemplate`(手动序列化)        |
| --------- | ------------------------------------------- | ------------------------------------------ |
| **序列化方式** | 由框架自动将Java对象与JSON字符串相互转换。                   | 开发者手动将对象转为JSON字符串，仅存储字符串。                  |
| **优点**    | 使用方便，代码简洁，类似操作原生Java对象。                     | **内存占用小**（无@class信息），**灵活性强**（可自由选择序列化工具）。 |
| **缺点**    | **有额外内存开销**（JSON中含@class信息），**可能存在类型安全风险**。 | 需编写手动序列化/反序列化代码，稍显繁琐。                      |
| **适用场景**  | 对开发效率要求高、数据量不大、内存不敏感的内部应用。                  | **生产环境推荐方案**，尤其关注内存效率和性能的场景。               |

---

### 方案详解与实践

#### 1. 为何要避免默认的JDK序列化？

如**图1**所示，默认的`RedisTemplate`使用JDK序列化，存在显著问题：

- **可读性差**：存储的键和值变为不可读的二进制格式（如`\xAC\xED\x00\x05t\x00\x04name`）。
    
- **内存占用大**：序列化后的字节流比较冗长。
    
- **跨平台性差**：严重依赖Java类，其他语言客户端无法读取。
    

#### 2. 方案一：自定义配置 `RedisTemplate`

这是**第一种改良方案**，通过配置Bean来替换默认的序列化器。

```
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(redisConnectionFactory);

    // 设置Key的序列化器为String序列化器
    template.setKeySerializer(RedisSerializer.string());
    template.setHashKeySerializer(RedisSerializer.string());

    // 设置Value的序列化器为JSON序列化器
    GenericJackson2JsonRedisSerializer jsonSerializer = new GenericJackson2JsonRedisSerializer();
    template.setValueSerializer(jsonSerializer);
    template.setHashValueSerializer(jsonSerializer);

    return template;
}
```

**配置后**，存储的Value会是清晰的JSON格式。但这会引入新问题：为了在反序列化时能确定对象类型，JSON中会包含类的全限定名（如`"@class":"com.heima.redis.pojo.User"`），导致**额外的内存开销**。

#### 3. 方案二（生产环境推荐）：使用 `StringRedisTemplate`+ 手动序列化

这是**更优的实践**，为了节省内存，我们统一使用String序列化器。

- **核心思想**：只存储纯粹的字符串（String类型的key和value）。将对象的序列化（Java Object -> JSON String）和反序列化（JSON String -> Java Object）过程交由开发者手动完成。
    
- **实现方式**：Spring已经提供了一个开箱即用的`StringRedisTemplate`类，它的key和value序列化方式默认就是String方式。
    

**完整的代码示例：

```
@Autowired
private StringRedisTemplate stringRedisTemplate; // 注入Spring提供的模板

// JSON工具（如Jackson的ObjectMapper）
private static final ObjectMapper mapper = new ObjectMapper();

@Test
void testSaveUser() throws JsonProcessingException {
    // 1. 创建Java对象
    User user = new User("虎哥", 21);
    // 2. 手动序列化为JSON字符串
    String json = mapper.writeValueAsString(user);
    // 3. 写入Redis（存储的是纯字符串）
    stringRedisTemplate.opsForValue().set("user:200", json);

    // 4. 从Redis读取数据（得到的是纯字符串）
    String jsonUser = stringRedisTemplate.opsForValue().get("user:200");
    // 5. 手动反序列化回Java对象
    User user1 = mapper.readValue(jsonUser, User.class);
    System.out.println("user1 = " + user1);
}
```

