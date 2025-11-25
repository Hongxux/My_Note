---
aliases:
  - 计量单位绑定
---
在 Spring Boot 配置中，合理使用计量单位能让配置项的含义清晰明确，避免歧义。其核心是借助 JDK 8 引入的 `Duration`（时间间隔）和 `DataSize`（数据大小）类型，并提供了两种简洁的方式来配置它们。

1. **首选配置文件带单位写法**：在配置文件中直接写明单位（如 `30s`, `10MB`）比使用注解指定单位更具可读性，能让人一目了然。
    
2. **确保使用 `@ConfigurationProperties`**：上述的单位自动转换功能是 **`@ConfigurationProperties`注解提供的**。如果你使用 `@Value("${some.property}")`注解来注入单个属性，Spring Boot 将**不会**自动进行单位转换，你需要自己处理字符串到 `Duration`或 `DataSize`类型的解析。
    
3. **注解与配置的优先级**：如果同时在配置文件里写了单位（如 `10MB`），又在字段上使用了 `@DataSizeUnit`注解（如 `DataUnit.GIGABYTES`），**配置文件中明确的单位具有更高优先级**。
### ⏳ 配置时间间隔 (Duration)

`Duration`类型用于表示时间间隔。Spring Boot 为其提供了灵活的支持。

|配置方式|示例 (YAML)|说明|
|---|---|---|
|**单位后缀**(推荐)|`timeout: 30s`|在值后直接加单位，如 `s`代表秒。|
|**注解指定**|`@DurationUnit(ChronoUnit.MINUTES)`|在字段上使用注解指定单位。|

**单位后缀示例**

在配置文件中直接书写带单位的值是最常见和直观的方式。Spring Boot 支持的时间单位非常丰富：

```
my-app:
  session-timeout: 30m    # 30分钟
  read-delay: 500ms       # 500毫秒
  default-wait: 10s       # 10秒
```

对应的配置类如下：

```
@ConfigurationProperties(prefix = "my-app")
@Data
public class MyAppProperties {
    private Duration sessionTimeout;
    private Duration readDelay;
    private Duration defaultWait;
}
```

**注解指定示例**

你也可以在 Java 类中明确指定单位，这样配置文件中就只需写数字：

```
my-app:
  backup-interval: 24
```

```
@ConfigurationProperties(prefix = "my-app")
@Data
public class MyAppProperties {
    @DurationUnit(ChronoUnit.HOURS) // 声明单位是“小时”
    private Duration backupInterval;
}
```

### 💾 配置数据大小 (DataSize)

`DataSize`类型用于表示数据量大小，其使用方式与 `Duration`类似。

|配置方式|示例 (YAML)|说明|
|---|---|---|
|**单位后缀**(推荐)|`max-file-size: 10MB`|在值后直接加单位，如 `MB`代表兆字节。|
|**注解指定**|`@DataSizeUnit(DataUnit.GIGABYTES)`|在字段上使用注解指定单位。|

**单位后缀示例**

这是最直接的方式，支持从 `B`(字节) 到 `TB`(太字节) 的单位：

```
upload:
  max-size: 10MB
  chunk-size: 512KB
```

对应的配置类如下：

```
@ConfigurationProperties(prefix = "upload")
@Data
public class UploadProperties {
    private DataSize maxSize;
    private DataSize chunkSize;
}
```

**注解指定示例**

同样，也可以通过注解在代码中固定单位：

```
app:
  cache-size: 2
```

```
@ConfigurationProperties(prefix = "app")
@Data
public class AppProperties {
    @DataSizeUnit(DataUnit.GIGABYTES) // 声明单位是“GB”
    private DataSize cacheSize;
}
```

