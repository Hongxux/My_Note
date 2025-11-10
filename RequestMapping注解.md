---
aliases:
  - RequestMapping
---

### 核心作用与映射位置

简单来说，`@RequestMapping`注解的主要作用是**建立 HTTP 请求与处理该请求的控制器方法之间的映射关系**。

- **工作原理**：当 Spring MVC 框架的核心控制器 `DispatcherServlet`接收到一个 HTTP 请求时，它会根据请求的 URL、方法类型等信息，去查找被 `@RequestMapping`及其衍生注解（如 `@GetMapping`）标记的方法。找到匹配的方法后，便会调用它来处理请求并生成响应 。
    
- **标注位置**：此注解可以标注在**类**上或**方法**上，二者通常配合使用，形成一种层级关系 。
    
    - **在类上**：提供请求路径的**公共前缀**，相当于一个模块的根路径。例如，`@RequestMapping("/users")`表示该类下所有方法的访问路径都以 `/users`开头 。
        
    - **在方法上**：提供具体的**路径映射**，追加在类级别的路径之后。例如，在类级别注解为 `/users`的类中，一个方法上的 `@RequestMapping("/profile")`映射的完整访问路径就是 `/users/profile`。
        
    

### 主要属性详解

`@RequestMapping`提供了多个属性，允许您对映射条件进行精细控制，确保只有完全符合条件的请求才会触发相应的方法 。

| 属性名                 | 作用               | 示例与说明                                                                                                                           |
| ------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| **`value`/ `path`** | **映射请求路径**（最常用）  | `@RequestMapping(value="/detail")`或 `@RequestMapping(path="/detail")`。支持**Ant风格通配符**（如 `?`匹配一个字符，`*`匹配多层路径）和**路径变量**（`/{id}`） 。 |
| **`method`**        | **限制HTTP请求方法**   | `@RequestMapping(value = "/create", method = RequestMethod.POST)`。表示该方法仅处理 POST 请求 。                                            |
| **`params`**        | **要求必须包含特定请求参数** | `@RequestMapping(..., params = "type=admin")`。表示请求中必须包含名为 `type`且值等于 `admin`的参数 。                                               |
| **`headers`**       | **要求必须包含特定请求头**  | `@RequestMapping(..., headers = "Content-Type=application/json")`。通常用于限制 AJAX 请求或特定内容类型 。                                       |
| **`consumes`**      | **指定处理请求的媒体类型**  | `@RequestMapping(..., consumes = "application/json")`。表示该方法仅处理请求头中 `Content-Type`为 `application/json`的请求 。                      |
| **`produces`**      | **指定返回响应的媒体类型**  | `@RequestMapping(...,produces = "application/json")`。表示该方法产生并返回 `application/json`类型的数据 。                                       |




#### [[Spring 注解中多值/条件属性的统一语法总结]]


#### 1. value / path 属性（核心路径映射）

这是 `@RequestMapping`**最常用**的属性，用于定义请求的 URL 路径模式。

##### 基本用法

```
// 精确匹配
@RequestMapping(value = "/users")
public String getUsers() {
    return "userList";
}

// path 是 value 的别名，功能完全相同
@RequestMapping(path = "/products")
public String getProducts() {
    return "productList";
}
```

##### 支持 Ant 风格通配符

Spring MVC 支持强大的 Ant 风格模式匹配：

```
@Controller
@RequestMapping("/api")
public class AdvancedMappingController {
    
    // ? 匹配单个字符
    @RequestMapping("/user?") // 匹配: /api/user1, /api/userA, 但不匹配 /api/user12
    public String singleChar() {
        return "single";
    }
    
    // * 匹配零个或多个字符（单级路径）
    @RequestMapping("/cate*") // 匹配: /api/cate, /api/category, /api/cate123
    public String wildcard() {
        return "wildcard";
    }
    
    // ** 匹配零个或多个路径（多级路径）
    @RequestMapping("/files/**") // 匹配: /api/files/, /api/files/doc, /api/files/images/2024/photo.jpg
    public String multiLevel() {
        return "files";
    }
    
    // 组合使用
    @RequestMapping("/pro?uct*/**") // 灵活的模式匹配
    public String combination() {
        return "combination";
    }
}
```

##### 路径变量（Path Variables）

这是 RESTful API 的核心特性：

```
@RestController
public class UserController {
    
    // 基本路径变量
    @RequestMapping("/users/{userId}") // {userId} 是占位符
    public String getUserById(@PathVariable("userId") Long id) { // 使用 @PathVariable 获取值
        return "User ID: " + id; // 访问 /users/123 → "User ID: 123"
    }
    
    // 路径变量名可省略（当方法参数名与占位符名相同时）
    @RequestMapping("/products/{productId}/categories/{categoryId}")
    public String getProductInCategory(@PathVariable Long productId, 
                                       @PathVariable String categoryId) {
        return "Product " + productId + " in category " + categoryId;
    }
    
    // 正则表达式约束路径变量
    @RequestMapping("/users/{userId:\\d+}") // 只匹配数字
    public String getUserWithRegex(@PathVariable String userId) {
        return "User with numeric ID: " + userId;
    }
}
```

##### 多路径映射

一个方法可以映射到多个路径：

```
@RequestMapping({"/home", "/index", "/main"}) // 数组形式，匹配多个路径
public String homePage() {
    return "home";
}
```

#### 2. method 属性（HTTP 方法限制）

限制只处理特定的 HTTP 请求方法，是构建 RESTful API 的基础。

##### 基本用法

```
@Controller
@RequestMapping("/api/books")
public class BookController {
    
    // 只处理 GET 请求（获取数据）
    @RequestMapping(method = RequestMethod.GET)
    public String getAllBooks() {
        return "bookList";
    }
    
    // 只处理 POST 请求（创建资源）
    @RequestMapping(method = RequestMethod.POST)
    public String createBook() {
        return "createSuccess";
    }
    
    // 组合使用：路径 + 方法
    @RequestMapping(value = "/{id}", method = RequestMethod.GET)
    public String getBookById(@PathVariable Long id) {
        return "bookDetail";
    }
    
    @RequestMapping(value = "/{id}", method = RequestMethod.PUT)
    public String updateBook(@PathVariable Long id) {
        return "updateSuccess";
    }
    
    @RequestMapping(value = "/{id}", method = RequestMethod.DELETE)
    public String deleteBook(@PathVariable Long id) {
        return "deleteSuccess";
    }
}
```

##### 推荐使用组合注解

为了代码更简洁，Spring 提供了专门的方法注解：

```
// 等价于 @RequestMapping(method = RequestMethod.GET)
@GetMapping
public String getAllBooks() { return "bookList"; }

// 等价于 @RequestMapping(method = RequestMethod.POST)
@PostMapping  
public String createBook() { return "createSuccess"; }

// 等价于 @RequestMapping(value = "/{id}", method = RequestMethod.GET)
@GetMapping("/{id}")
public String getBookById(@PathVariable Long id) { return "bookDetail"; }

// PUT 和 DELETE 同样有对应注解
@PutMapping("/{id}")
@DeleteMapping("/{id}")
```

#### 3. params 属性（请求参数条件）

根据请求参数的存在性、值进行精确匹配。

##### 参数存在性检查

```
@Controller
@RequestMapping("/search")
public class SearchController {
    
    // 必须包含 keyword 参数（值任意）
    @RequestMapping(params = "keyword")
    public String searchWithKeyword() {
        return "searchResult";
    }
    
    // 必须不包含 category 参数
    @RequestMapping(params = "!category")
    public String searchWithoutCategory() {
        return "generalSearch";
    }
    
    // 必须同时包含 keyword 和 page 参数
    @RequestMapping(params = {"keyword", "page"})
    public String searchWithPagination() {
        return "pagedResults";
    }
    
    // 必须包含 keyword 但不能包含 debug 参数
    @RequestMapping(params = {"keyword", "!debug"})
    public String productionSearch() {
        return "productionResults";
    }
}
```

##### 参数值精确匹配

```
@Controller
@RequestMapping("/filter")
public class FilterController {
    
    // 必须包含 type=admin 参数
    @RequestMapping(params = "type=admin")
    public String adminView() {
        return "adminPage";
    }
    
    // 必须包含 status 参数，且值不能是 pending
    @RequestMapping(params = "status!=pending")
    public String nonPendingItems() {
        return "activeItems";
    }
    
    // 多参数值匹配
    @RequestMapping(params = {"role=manager", "dept=IT"})
    public String itManagers() {
        return "itManagerView";
    }
    
    // 复杂条件：有keyword参数，type不能是temp，必须有page但值不能是0
    @RequestMapping(params = {"keyword", "type!=temp", "page!=0"})
    public String advancedSearch() {
        return "advancedResults";
    }
}
```

##### 实际应用场景

```
@Controller
@RequestMapping("/products")
public class ProductController {
    
    // 普通商品列表
    @RequestMapping(params = "!sale") // 不包含 sale 参数
    public String regularProducts() {
        return "regularProducts";
    }
    
    // 特价商品列表
    @RequestMapping(params = "sale=true") // sale=true
    public String saleProducts() {
        return "saleProducts";
    }
    
    // 清仓商品（特价且库存紧张）
    @RequestMapping(params = {"sale=true", "clearance=true"})
    public String clearanceProducts() {
        return "clearanceProducts";
    }
}
```

#### 4. headers 属性（请求头条件）

根据 HTTP 请求头进行精确匹配，常用于 API 版本控制、内容协商等场景。

##### 请求头存在性检查

```
@Controller
@RequestMapping("/api")
public class ApiController {
    
    // 必须包含 X-API-Key 请求头
    @RequestMapping(headers = "X-API-Key")
    public String authenticatedApi() {
        return "authenticatedResponse";
    }
    
    // 必须不包含 Debug-Mode 请求头
    @RequestMapping(headers = "!Debug-Mode")
    public String productionApi() {
        return "productionResponse";
    }
    
    // 多头部要求
    @RequestMapping(headers = {"X-API-Key", "X-Client-Version"})
    public String versionedApi() {
        return "versionedResponse";
    }
}
```

##### 请求头值精确匹配

```
@RestController
public class ContentController {
    
    // 只接受 Content-Type=application/json 的请求
    @RequestMapping(headers = "Content-Type=application/json")
    public String handleJson() {
        return "jsonResponse";
    }
    
    // 只接受 Accept=application/xml 的请求
    @RequestMapping(headers = "Accept=application/xml")
    public String produceXml() {
        return "<response>XML</response>";
    }
    
    // API 版本控制
    @RequestMapping(headers = "API-Version=1")
    public String apiV1() {
        return "API Version 1";
    }
    
    @RequestMapping(headers = "API-Version=2")
    public String apiV2() {
        return "API Version 2";
    }
}
```

##### 实际应用场景

```
@RestController
@RequestMapping("/data")
public class DataController {
    
    // 移动端专用 API（通过 User-Agent 识别）
    @RequestMapping(headers = "User-Agent=MobileApp/1.0")
    public String mobileApi() {
        return "mobileOptimizedData";
    }
    
    // Web 端 API
    @RequestMapping(headers = "User-Agent=Mozilla*")
    public String webApi() {
        return "fullData";
    }
    
    // 只处理 AJAX 请求
    @RequestMapping(headers = "X-Requested-With=XMLHttpRequest")
    public String ajaxHandler() {
        return "ajaxResponse";
    }
}
```

##### 综合使用示例

在实际项目中，这些属性经常组合使用：

```
@RestController
@RequestMapping("/api/v1")
public class ComprehensiveController {
    
    // 精确的 RESTful 端点定义
    @RequestMapping(
        value = "/users/{id}",
        method = RequestMethod.GET,
        headers = {"Content-Type=application/json", "API-Version=1"},
        params = "fields=basic"
    )
    public User getBasicUserInfo(@PathVariable Long id) {
        // 只返回用户基本信息
        return userService.getBasicInfo(id);
    }
    
    @GetMapping(
        value = "/users/{id}",
        headers = "API-Version=2",
        params = "fields=detailed"
    )
    public User getDetailedUserInfo(@PathVariable Long id) {
        // 返回用户详细信息（API 版本 2）
        return userService.getDetailedInfo(id);
    }
}
```

#### 匹配优先级与冲突解决

当多个映射规则可能匹配同一个请求时，Spring MVC 按**最具体**的原则选择：

1. **路径模式更具体的优先**（如 `/users/123`比 `/users/*`更具体）
    
2. **参数/头部条件更多的优先**
    
3. **方法更具体的优先**（如 GET 比 无方法限制的更具体）
    

这种精细的匹配机制让 Spring MVC 能够处理复杂的路由需求，是构建健壮 Web 应用的基础。


### 实际应用与最佳实践

在实际项目中，`@RequestMapping`最常见的应用场景是构建 **RESTful API**。通过结合类级别和方法级别的路径，以及不同的 HTTP 方法，可以清晰地表达资源操作 。

```
@RestController
@RequestMapping("/api/users") // 类级别：定义API资源路径
public class UserRestController {

    @GetMapping // 对应：GET /api/users，获取用户列表
    public List<User> getUsers() { ... }

    @GetMapping("/{id}") // 对应：GET /api/users/1，获取ID为1的用户
    public User getUserById(@PathVariable Long id) { ... }

    @PostMapping // 对应：POST /api/users，创建新用户
    public User createUser(@RequestBody User user) { ... }

    @PutMapping("/{id}") // 对应：PUT /api/users/1，更新ID为1的用户
    public User updateUser(@PathVariable Long id, @RequestBody User user) { ... }

    @DeleteMapping("/{id}") // 对应：DELETE /api/users/1，删除ID为1的用户
    public void deleteUser(@PathVariable Long id) { ... }
}
```
