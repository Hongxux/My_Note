

|特性维度|**`@RequestParam`**|**`@PathVariable`**|**`@RequestBody`**|**无需注解（表单绑定）**|
|---|---|---|---|---|
|**核心用途**|获取 **URL查询字符串**中的参数|获取 **URL路径**中的变量部分|将 **请求体**中的数据转换为Java对象|将 **查询字符串或表单数据**自动绑定到Java对象|
|**传参方式**|`?key1=value1&key2=value2`|`/value`(如 `/users/123`)|请求体（如JSON、XML）|`?keyword=abc&page=1`(同`@RequestParam`，但自动封装)|
|**参数可选性**|默认必选，可设 `required=false`改为可选|默认必选，路径变量不可或缺|通常为必选（无请求体则请求不成立）|对象属性可选与否取决于字段本身|
|**主要特点**|1. 可处理单个、多个参数或Map  <br>2. 可设置默认值(`defaultValue`)|1. 专为RESTful风格设计  <br>2. 从URI模板中提取值|1. 通常用于POST/PUT请求  <br>2. 根据`Content-Type`（如`application/json`）进行转换|1. **无需任何注解**  <br>2. Spring自动匹配参数名到对象属性  <br>3. 支持嵌套属性|
|**适用HTTP方法**|GET, POST等所有方法|GET, POST, PUT, DELETE等所有方法|主要为POST, PUT|GET, POST|
|**简单示例**|`?name=John`|`/users/123`|`{"name":"Alice"}`|`?keyword=abc&page=1`|
### 一、 接收简单参数：`@RequestParam`

这是最常用的方式，主要用于获取 ​**URL 查询字符串（Query String）​**​ 中的参数。

- ​**适用场景**​：`/user?name=John&age=20`
    
- ​**注解**​：`@RequestParam`
    
- ​**参数可选性**​：默认情况下，参数是**必须的**。如果请求中缺失，会返回 400 错误。可以通过设置 `required = false`将其改为可选。
    

​**示例代码：​**​



1. 基本用法：获取单个参数
```
    // 1. 基本用法：获取单个参数
    // 访问：GET /user?name=John
    @GetMapping("/user")
    public String getUserByName(@RequestParam String name) {
        return "用户名: " + name;
    }

```
 2. 明确指定参数名，并设置为可选
```
    // 访问：GET /user?name=John&age=20 或 GET /user?name=John
    @GetMapping("/user")
    public String getUser(
            @RequestParam("name") String userName, // 将请求中的"name"映射到userName变量
            @RequestParam(value = "age", required = false) Integer age) { // age是可选的
        if (age != null) {
            return "用户名: " + userName + ", 年龄: " + age;
        }
        return "用户名: " + userName;
    }
```


3. 接收所有参数到一个Map或对象
```
    // 访问：GET /user?name=John&age=20&city=Beijing
    @GetMapping("/user")
    public String getUserByMap(@RequestParam Map<String, Object> params) {
        return "参数: " + params.toString(); // 输出: {name=John, age=20, city=Beijing}
    }
}
```




### 二、 接收路径参数：`@PathVariable`

用于获取 ​**URL 路径**​ 中的变量部分，常用于 RESTful 风格的 API。

- ​**适用场景**​：`/user/1`，其中 `1`是用户的 ID。
    
- ​**注解**​：`@PathVariable`
    

​**示例代码：​**​

```
@RestController
public class UserController {

    // 1. 基本用法：从路径中提取变量
    // 访问：GET /user/123
    @GetMapping("/user/{id}")
    public String getUserById(@PathVariable Long id) {
        return "用户ID: " + id; // 输出: 用户ID: 123
    }

    // 2. 路径变量名与方法参数名不同时，需指定
    // 访问：GET /product/phone-12345
    @GetMapping("/product/{productId}")
    public String getProduct(@PathVariable("productId") String id) {
        return "产品ID: " + id; // 输出: 产品ID: phone-12345
    }

    // 3. 多个路径变量
    // 访问：GET /users/1/orders/1001
    @GetMapping("/users/{userId}/orders/{orderId}")
    public String getOrder(
            @PathVariable Long userId,
            @PathVariable Long orderId) {
        return "用户" + userId + "的订单：" + orderId;
    }
}
```

### 三、 接收JSON请求体：@RequestBody

用于接收 ​**HTTP 请求体（Body）​**​ 中的数据，通常用于 POST 或 PUT 请求，接收 JSON 或 XML 格式的数据，并将其转换为 Java 对象。

- ​**适用场景**​：前端传递一个 JSON 对象来创建或更新资源。
    
- ​**注解**​：`@RequestBody`
    
- ​**重要**​：通常与 `@PostMapping`或 `@PutMapping`一起使用。
    

​**示例代码：​**​

首先，定义一个与 JSON 结构对应的 Java 类（POJO）：

```
// Lombok 注解，自动生成 getter, setter, 无参构造函数等
@Data
public class User {
    private String name;
    private Integer age;
    private String email;
}
```

在控制器中接收：

```
@RestController
public class UserController {

    // 接收JSON，并自动映射到User对象
    // 请求：POST /user, Body: {"name": "Alice", "age": 25, "email": "alice@example.com"}
    @PostMapping("/user")
    public String createUser(@RequestBody User user) {
        // Spring Boot 会自动将JSON解析并填充到user对象的属性中
        return "创建用户: " + user.getName() + ", 邮箱: " + user.getEmail();
    }

    // 也可以接收为Map
    @PostMapping("/user-map")
    public String createUserByMap(@RequestBody Map<String, Object> userMap) {
        return "通过Map创建用户: " + userMap.get("name");
    }
}
```

### 四、 接收复杂嵌套对象（表单数据）

当提交的是表单数据（`application/x-www-form-urlencoded`）时，Spring Boot 也能自动将参数绑定到 Java 对象的属性上。​**这种情况下，通常不需要 `@RequestBody`**。

​**示例代码：​**​

```
@Data // 使用Lombok
public class QueryParams {
    private String keyword;
    private Integer page;
    private Integer size;
}

@RestController
public class SearchController {

    // 表单提交或URL参数会自动绑定到对象属性
    // 访问：GET /search?keyword=SpringBoot&page=1&size=10
    @GetMapping("/search")
    public String search(QueryParams params) { // 这里不需要任何注解！
        return "搜索关键词: " + params.getKeyword() + ", 第" + params.getPage() + "页";
    }
}
```

### 五、 其他有用注解

1. ​ `@RequestHeader` ：获取 HTTP 请求头信息。
    
    ```
    @GetMapping("/header")
    public String getHeader(@RequestHeader("User-Agent") String userAgent) {
        return "浏览器信息: " + userAgent;
    }
    ```
    
2. ​`@CookieValue` ​：获取 Cookie 值。
    
    ```
    @GetMapping("/cookie")
    public String getCookie(@CookieValue("JSESSIONID") String sessionId) {
        return "Session ID: " + sessionId;
    }
    ```
    

### 总结与最佳实践

|场景|传参方式|使用的注解|示例|
|---|---|---|---|
|​**查询参数**​|URL 问号后|`@RequestParam`|`/user?name=abc`|
|​**路径参数**​|URL 路径中|`@PathVariable`|`/user/123`|
|​**JSON 数据**​|请求体（Body）|`@RequestBody`|`POST /user`+ JSON Body|
|​**表单数据/对象**​|查询字符串或表单|​**无需注解**​（直接使用对象）|`GET /search?keyword=abc&page=1`|

​**核心要点：​**​

- ​**无注解绑定对象**​：当参数是一个自定义对象时，Spring Boot 会尝试自动将请求参数匹配到对象的属性上，这是一种非常简洁的方式。
    
- ​**Content-Type**​：使用 `@RequestBody`时，前端请求的 `Content-Type`通常应为 `application/json`。
    
- ​**组合使用**​：这些注解可以在一个方法中组合使用。
    
    ```
    @PostMapping("/users/{type}")
    public String complexExample(
            @PathVariable String type,          // 路径参数
            @RequestParam String action,        // 查询参数
            @RequestBody User user) {           // 请求体
        return "Type: " + type + ", Action: " + action + ", User: " + user.getName();
    }
    ```
    