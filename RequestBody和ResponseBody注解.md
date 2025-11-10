---
aliases:
  - RequestBody
  - ResponseBody
---
好的，`@RequestBody`和 `@ResponseBody`是 Spring MVC 中用于处理**请求数据绑定**和**响应数据渲染**的两个核心注解。它们是实现现代 RESTful Web 服务的关键。

下面我将为您详细讲解它们的作用、使用方法和最佳实践。

---

### 一、@RequestBody：将请求体绑定到方法参数

`@RequestBody`注解用于将 **HTTP 请求体**（如 JSON、XML 数据）中的数据，自动绑定（反序列化）到控制器方法的参数上。

#### 1. 核心作用

它告诉 Spring：“请从请求体（Body）中读取数据，并使用合适的**消息转换器**（如 `HttpMessageConverter`）将其转换为指定的 Java 对象类型。”

#### 2. 使用场景

主要用于处理 `POST`、`PUT`、`PATCH`等请求，这些请求通常通过请求体（如 JSON 格式）来传递复杂的结构化数据。

#### 3. 如何使用

在控制器方法的参数前添加 `@RequestBody`注解。

```
@RestController // @RestController = @Controller + @ResponseBody
@RequestMapping("/api/users")
public class UserController {

    // 经典用法：接收 JSON 数据并转换为 Java 对象
    @PostMapping
    public User createUser(@RequestBody User user) {
        // Spring 会自动将请求体中的 JSON 数据（如 {"name": "John", "email": "john@example.com"}）
        // 转换为 User 对象
        return userService.save(user);
    }

    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        // 结合路径变量和请求体数据
        user.setId(id);
        return userService.update(user);
    }
}
```

#### 4. 配合的请求

客户端（如 Postman 或前端）需要设置正确的 `Content-Type`请求头，通常是 `application/json`。

**HTTP 请求示例：**

```
POST /api/users HTTP/1.1
Host: localhost:8080
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

#### 5. 工作原理

1. 客户端发送请求，请求头中包含 `Content-Type: application/json`。
    
2. Spring 的 `DispatcherServlet`接收到请求。
    
3. 根据 `Content-Type`，Spring 选择匹配的 `HttpMessageConverter`（例如，`MappingJackson2HttpMessageConverter`用于处理 JSON）。
    
4. 转换器读取请求体（Body）中的原始数据，并将其反序列化为方法参数声明的 Java 对象（如 `User`对象）。
    
5. 调用控制器方法，并传入已填充数据的对象。
    

---

### 二、@ResponseBody：将方法返回值写入响应体

`@ResponseBody`注解用于将控制器方法的返回值直接写入 **HTTP 响应体**中，而不是交给视图解析器去寻找一个物理视图（如 JSP、Thymeleaf）。

#### 1. 核心作用

它告诉 Spring：“请不要将返回值解析为视图名称，而是使用合适的**消息转换器**将其直接序列化后写入响应输出流。”

#### 2. 使用场景

用于构建 RESTful API，直接向客户端返回数据（如 JSON、XML），而不是跳转到某个 HTML 页面。

#### 3. 如何使用

在方法上或返回值前添加 `@ResponseBody`注解。

```
@Controller
@RequestMapping("/api/products")
public class ProductController {

    // 在方法上使用 @ResponseBody
    @GetMapping("/{id}")
    @ResponseBody
    public Product getProduct(@PathVariable Long id) {
        // 返回值 Product 对象将被直接转换为 JSON 写入响应体
        return productService.findById(id);
    }

    @GetMapping
    @ResponseBody
    public List<Product> getAllProducts() {
        // 返回集合也会被转换为 JSON 数组
        return productService.findAll();
    }
}
```

#### 4. 便捷的替代方案：@RestController

在类级别使用 `@RestController`注解，相当于为类中的**所有方法**都自动添加了 `@ResponseBody`注解。这是 Spring 4.0 后推荐的做法，非常简洁。

```
@RestController // 等价于 @Controller + @ResponseBody on every method
@RequestMapping("/api/products")
public class ProductController {

    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        // 无需再写 @ResponseBody，返回值自动转为 JSON
        return productService.findById(id);
    }
}
```

#### 5. 配合的响应

Spring 会根据请求的 `Accept`头或默认配置，设置响应的 `Content-Type`，如 `application/json`。

**HTTP 响应示例：**

```
HTTP/1.1 200 OK
Content-Type: application/json

{
  "id": 1,
  "name": "Laptop",
  "price": 999.99
}
```

---

### 三、组合使用：完整的 RESTful 端点

一个典型的创建资源并返回创建结果的 RESTful 端点会同时使用这两个注解（或使用 `@RestController`）。

```
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @PostMapping
    public Order createOrder(@RequestBody Order order) {
        // @RequestBody：将请求体JSON -> Order对象
        // @ResponseBody（隐含）：将返回的Order对象 -> 响应体JSON
        return orderService.create(order);
    }
}
```

**客户端交互流程：**

1. **客户端发送（POST）**：
    
    ```
    {
      "items": [...],
      "totalAmount": 150.00
    }
    ```
    
2. **服务器处理**：`@RequestBody`将 JSON 转换为 `Order`对象。
    
3. **服务器返回**：`@ResponseBody`（由 `@RestController`隐含提供）将保存后的 `Order`对象转换为 JSON。
    
4. **客户端接收**：
    
    ```
    {
      "id": 101,
      "items": [...],
      "totalAmount": 150.00,
      "status": "CREATED"
    }
    ```
    

---

### 四、总结与对比

|特性|`@RequestBody`|`@ResponseBody`|
|---|---|---|
|**作用位置**|**方法参数**前|**方法**上或**返回值**前|
|**核心功能**|**反序列化**：将**请求体**数据转换为 Java 对象|**序列化**：将 Java 对象转换为**响应体**数据|
|**常用场景**|接收 `POST`/`PUT`请求的 JSON/XML 数据|返回 JSON/XML 数据，构建 API|
|**HTTP 方向**|**输入**（Input）|**输出**（Output）|
|**常用配对**|`@PostMapping`, `@PutMapping`|`@GetMapping`, `@PostMapping`, `@RequestMapping`|
|**简化注解**|无|`@RestController`（类级别）|

**最佳实践建议：**

- 开发 **RESTful API**时，直接在类上使用 `@RestController`，无需再为每个方法写 `@ResponseBody`。
    
- 确保项目中引入了 JSON 处理库（如 Jackson），Spring Boot 会自动配置。
    
- 客户端请求时应设置正确的 `Content-Type`头（如 `application/json`）。
    

掌握这两个注解，你就掌握了使用 Spring MVC 构建现代数据驱动 Web 应用和 API 服务的核心技能。