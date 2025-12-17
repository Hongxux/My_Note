
**实现目标**：AOP实现**声明式缓存**：你只需在方法上添加一个注解（如`@MyCache`），缓存逻辑（查缓存、未命中则查DB、回填缓存）就会自动生效。

#### 1. 定义缓存注解

首先，创建一个自定义注解。它作为“标记”，告诉AOP框架哪些方法需要被增强。

```
@Target(ElementType.METHOD) // 该注解只能用在方法上
@Retention(RetentionPolicy.RUNTIME) // 注解在运行时生效
public @interface MyCache {
    String keyPrefix(); // 缓存键的前缀，如"user"
    String matchValue() default ""; // 支持SpEL表达式，动态生成缓存键，如"#id"
    long expireTime() default 3600; // 缓存过期时间，默认1小时
}
```

#### 2. 实现缓存切面

这是AOP的核心，切面（Aspect）包含了具体的缓存逻辑。

```
@Aspect // 声明这是一个切面类
@Component
@Slf4j
public class CacheAspect {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    // 定义切点：所有被@MyCache注解标记的方法
    @Pointcut("@annotation(com.yourpackage.annotation.MyCache)")
    public void cachePointcut() {}

    // 环绕通知：在目标方法执行前后执行
    @Around("cachePointcut()")
    public Object doCache(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        MyCache myCache = method.getAnnotation(MyCache.class);

        // 1. 动态生成唯一的缓存Key（例如："user:123"）
        String cacheKey = generateCacheKey(myCache, joinPoint);

        // 2. 先查缓存
        Object cacheValue = redisTemplate.opsForValue().get(cacheKey);
        if (cacheValue != null) {
            log.info("缓存命中，key: {}", cacheKey);
            return cacheValue; // 命中则直接返回，不再执行原方法
        }

        // 3. 缓存未命中，执行原方法（即查询数据库）
        log.info("缓存未命中，查询数据库，key: {}", cacheKey);
        Object result = joinPoint.proceed();

        // 4. 将数据库查询结果回写到缓存中
        if (result != null) {
            redisTemplate.opsForValue().set(cacheKey, result, myCache.expireTime(), TimeUnit.SECONDS);
        }
        return result;
    }

    private String generateCacheKey(MyCache myCache, ProceedingJoinPoint joinPoint) {
        // 使用SpEL解析注解中的matchValue，将其转换为方法参数的实际值
        // 例如，matchValue = "#id", 方法参数id=123, 则生成 "user:123"
        // 具体实现可参考Spring的SpEL表达式解析
        return myCache.keyPrefix() + ":" + ... ;
    }
}
```

#### 3. 在业务层使用

现在，你的业务代码变得非常简洁，只需关注核心逻辑。

```
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private UserMapper userMapper;

    @Override
    @MyCache(keyPrefix = "user", matchValue = "#id", expireTime = 1800) // 声明式缓存
    public UserDTO getUserById(Long id) {
        // 只需专注于从数据库查询用户，缓存逻辑由AOP自动处理
        return userMapper.selectById(id);
    }

    @Override
    @Transactional
    public void updateUser(UserDTO user) {
        // 更新数据库
        userMapper.updateById(user);
        // 注意：对于写操作，通常还需要有删除缓存的逻辑，这同样可以通过AOP注解（如@CacheEvict）实现
    }
}
```

通过AOP，缓存逻辑被集中到了`CacheAspect`中。如果需要修改缓存策略（比如引入分布式锁防止缓存击穿），只需修改切面即可，所有使用了`@MyCache`注解的方法都会自动生效，极大提升了可维护性。

