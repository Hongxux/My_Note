---
aliases:
  - Controller
---
在标准的企业级应用开发中，Controller层作为应用的 **“门面”**，其核心设计思想是遵循 **“瘦控制器”（Thin Controller）** 原则，即**使其职责单一、简洁高效，不包含任何业务逻辑**，只负责协调和委托工作。
搭建框架：
1. 类上[[@RestController]]注解和@RequestMapping("xxx/xxx")注解
2. 方法上@PostMapping @GetMapping等等符合[[RESTFul编程风格|RESTFul]]开发风格




主要负责：
1. [[SpringBoot接受请求参数|接受请求信息]]
2. 调用对应Service层的方法，并且传递请求的信息
3. 返回Service层的方法返回的[[Result]]（数据封装）
   
为了帮你快速建立整体认知，下表清晰地概括了其核心职责与设计要点。

| 核心职责           | 设计要点与最佳实践                                                     |
| -------------- | ------------------------------------------------------------- |
| **请求路由与解析**    | 接收HTTP请求，解析参数（路径、查询、请求体等），并完成基本校验。                            |
| **参数校验**       | 使用JSR-303注解（如`@Valid`, `@NotBlank`)进行声明式校验，使代码简洁且错误信息明确。      |
| **调用Service层** | **委托业务逻辑**：将业务处理委托给Service层，自身不包含业务规则。                        |
| **统一响应封装**     | 确保返回统一的响应格式（如包含code, message, data的JSON对象），提升客户端处理效率。         |
| **异常处理**       | 通常结合全局异常处理（`@RestControllerAdvice`）来捕获和处理异常，保持Controller方法清爽。 |
| **转换业务对象**     | 进行简单的DTO（Data Transfer Object）与内部业务对象之间的转换，隔离外部接口与内部模型。       |

下面，我们深入探讨如何将这些思想付诸实践。

### 🔩 核心实践与代码示例

#### 1. 保持“瘦控制器”，委托业务逻辑

Controller应作为协调者，其核心工作是调用一个或几个Service方法，而不是自己实现业务规则。

```
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @Autowired
    private UserService userService; // 委托业务逻辑给Service

    @PostMapping
    public ResponseEntity<UserDTO> createUser(@Valid @RequestBody UserCreateRequest request) {
        // 参数校验已由 @Valid 处理，Controller只负责调用Service并返回结果
        UserDTO createdUser = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdUser);
    }
}
```

#### 2. 使用声明式参数校验

在接收参数的DTO类字段上使用校验注解，并在Controller方法参数前添加`@Valid`注解，可以极大简化代码并提高可读性。

```
@Data
public class UserCreateRequest {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度必须在2-20个字符之间")
    private String username;

    @Email(message = "邮箱格式不正确")
    private String email;
}

// 在Controller中使用
@PostMapping
public ApiResponse<UserDTO> createUser(@Valid @RequestBody UserCreateRequest request) {
    // ... 调用service
}
```

#### 3. 返回统一的响应格式

定义统一的响应体（如`ApiResponse`），便于前端处理，并能统一处理成功、失败、异常等情况。

```
// 统一的响应封装
@Data
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp = System.currentTimeMillis();

    // 成功的静态工厂方法
    public static <T> ApiResponse<T> success(T data) {
        ApiResponse<T> response = new ApiResponse<>();
        response.setCode(200);
        response.setMessage("success");
        response.setData(data);
        return response;
    }
}

// Controller使用统一响应
@GetMapping("/{id}")
public ApiResponse<UserDTO> getUserById(@PathVariable Long id) {
    UserDTO user = userService.getUserById(id);
    return ApiResponse.success(user); // 统一格式
}
```

#### 4. 实现全局异常处理

使用`@RestControllerAdvice`或`@ControllerAdvice`进行全局异常处理，避免在Controller中使用大量的try-catch代码块。

```
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 处理参数校验异常
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ApiResponse<Object> handleValidationException(MethodArgumentNotValidException ex) {
        String errorMessage = ex.getBindingResult().getFieldErrors().stream()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .collect(Collectors.joining("; "));
        return ApiResponse.fail(400, errorMessage);
    }

    // 处理业务异常
    @ExceptionHandler(BusinessException.class)
    public ApiResponse<Object> handleBusinessException(BusinessException ex) {
        return ApiResponse.fail(ex.getCode(), ex.getMessage());
    }
}
```

配置全局异常处理后，Controller方法将非常简洁，只需关注正常流程。

### 💡 高级特性与最佳实践

1. **RESTful API设计**：遵循REST架构风格，正确使用HTTP方法（GET-查询, POST-创建, PUT-更新, DELETE-删除）和资源命名。
    
2. **API版本控制**：建议将版本号（如`/api/v1/`）放入URL路径，为后续接口迭代留出空间。
    
3. **使用切面（AOP）**：对于跨关注点的功能，如日志记录、性能监控、权限检查等，建议使用Spring AOP实现，避免代码分散。
    
4. **编写测试**：对Controller层进行单元测试（如使用`@WebMvcTest`）和集成测试，确保接口按预期工作。
    

### 💎 总结

标准的Controller层设计思想，精髓在于 **“各司其职”**。它应如一个高效的交通指挥，只负责请求的接收、调度和响应的发出，而将复杂的“交通运输”（业务逻辑）交给Service层。通过遵循上述原则与实践，你可以构建出**职责清晰、易于维护、稳健可靠**的API接口，为打造高质量的应用程序奠定坚实基础。

希望这些讲解和示例能帮助你更好地理解和实践Controller层的标准开发思想。