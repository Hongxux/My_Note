 1. 定义控制器与路由映射
	- **使用 `@RestController`注解**：这个注解是 `@Controller`和 `@ResponseBody`的组合，表明该类是一个控制器，并且所有方法返回的数据（如对象、集合）会直接写入HTTP响应体，通常被转换为JSON格式，适用于开发RESTful API。
	- **使用 `@RequestMapping`及其变体**：这些注解用于将特定的URL路径映射到控制器方法上。
	    - **`@RequestMapping`**：可以定义在类级别（为所有方法设置统一路径前缀）和方法级别，并能指定HTTP方法（如 `method = RequestMethod.GET`）。
	    - **快捷注解**：更推荐使用 `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`等，它们专门用于处理特定类型的HTTP请求，使代码更简洁。
 2. 处理请求参数：Controller需要灵活地获取客户端传递过来的各种数据
	- **路径变量：`@PathVariable`**：用于从URL路径模板中提取变量值。例如，`/users/{id}`中的 `{id}`可以通过 `@PathVariable Long id`绑定到方法参数上。
	- **查询参数：`@RequestParam`**：主要用于获取URL问号（?）后面的参数。你可以设置参数是否必须（`required`属性）和默认值（`defaultValue`属性）。
	- **请求体：`@RequestBody`**：当客户端提交JSON或XML等格式的复杂数据时，使用此注解可以将请求体内容直接映射到一个Java对象上。
	- **其他参数**：还可以通过 `@RequestHeader`获取请求头信息，或者直接使用 `HttpServletRequest`对象来获得更全面的请求数据。
3. 构建与返回响应：Controller方法需要清晰地定义返回给客户端的内容和状态。
	- **统一响应格式**：为了便于前端处理，建议所有接口返回统一的JSON数据格式。可以定义一个通用的包装类（如 `ResultBean`或 `R<T>`），包含状态码（`code`）、消息（`message`）和实际数据（`data`）等字段。这样做不仅使返回结果标准化，也方便进行全局异常处理和日志记录。
	- **控制HTTP状态码**：使用 `@ResponseStatus`注解可以显式地指定方法返回的HTTP状态码，例如在创建资源成功后返回 `201 Created`。
	- **明确返回必要信息**：例如，创建资源的接口通常应返回新创建的对象信息或至少其ID，而不是只返回一个简单的布尔值。
 4. 调用Service层与依赖注入：Controller层应专注于协调请求和响应，而将具体的业务逻辑委托给Service层处理。
	- **依赖注入**：通过 `@Autowired`或构造函数注入的方式（推荐使用构造函数注入，利于不可变性和测试）将Service层的实例引入Controller。这样符合分层架构的原则，使代码更清晰，易于测试和维护。
	- **保持Controller层轻薄**：Controller方法本身不应该包含复杂的业务规则或数据访问逻辑。它的主要职责是调用Service层的方法，并根据返回结果组织响应。
  5. 处理异常：一个健壮的系统需要妥善处理可能出现的异常情况。
	- **使用全局异常处理**：虽然可以在Controller方法内部使用 `try-catch`，但更优雅和高效的方式是采用Spring的全局异常处理机制（如使用 `@ControllerAdvice`或 `@RestControllerAdvice`注解的类）。这样可以集中处理各种异常，避免在每一个Controller方法中编写重复的异常处理代码，保证返回给客户端的错误信息也是统一的格式。
    