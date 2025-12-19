
-  需求背景：传统 Servlet 开发的弊端*
	传统 Java Web 开发基于 Servlet API，但存在以下核心问题，这些弊端促使了 Spring MVC 框架的出现：
	- **代码冗余与配置繁琐**：每个请求需创建独立的 `Servlet`并重写 `doGet`/`doPost`方法，导致大量重复代码。此外，Servlet 需在 `web.xml`中手动注册，管理成本高。
	- **手动参数处理效率低**：请求参数需通过 `HttpServletRequest`的 `getParameter()`手动提取和类型转换，易出错且缺乏标准化。
	- **组件耦合性高**：JSP 与 JavaBean 强耦合，前后端逻辑混杂（如 JSP 内嵌 Java 代码），导致可维护性差、测试困难，且违反分层原则。
	- **缺乏统一请求入口**：每个 Servlet 独立处理请求，无法集中处理公共逻辑（如鉴权、日志），重复代码多。
	- **框架集成困难**：与 Spring IoC 容器等生态系统集成需手动配置，难以利用依赖注入等优势。
-  解决措施：Spring MVC 框架的核心机制**​ 
	- **注解驱动开发**：使用 `@Controller`、`@RequestMapping`等注解将 POJO 类转为控制器，减少 XML 配置，提升灵活性。
	- **自动化参数绑定**：通过 `@RequestParam`、`@PathVariable`、`@RequestBody`等注解自动提取和转换请求参数，支持数据验证和类型转换。
	- **职责分离的 MVC 架构**：
	    - **模型（Model）**：封装业务数据（POJO 或 `Model`对象），由 Service 层处理逻辑。
	    - **视图（View）**：模板引擎（如 JSP、Thymeleaf）负责渲染，与数据解耦。
	    - **控制器（Controller）**：处理请求、调用业务逻辑，返回视图名或数据。
	- **前端控制器模式**：`DispatcherServlet`作为统一入口，协调组件执行请求映射、处理器适配、视图解析等流程。
	- **无缝集成 Spring 生态**：天然支持依赖注入（IoC）、AOP、Spring Security 等，提升模块可测试性和可维护性。
 -  核心设计思想
	-  职责分离（MVC 模式）​
		- **目的**：实现高内聚、低耦合，使组件职责单一（控制器处理请求、模型封装数据、视图负责渲染）。
		- **协作机制**：
		    - 控制器接收请求后，调用 Service 层业务逻辑，并将数据存入模型。
		    - 视图根据模型数据渲染响应，避免业务逻辑渗透。
		- **作用**：提升代码可复用性、简化单元测试（如控制器可独立于视图测试）。
	 - 前端控制器模式​
		- **实现机制**：`DispatcherServlet`是所有请求的入口，其核心方法 `doDispatch()`协调九大组件（如 `HandlerMapping`、`ViewResolver`）。
		- **作用**：集中处理公共逻辑（如安全校验），降低组件依赖。
	- 高扩展性设计​
		- **可插拔组件**：通过接口（如 `HandlerInterceptor`、`ViewResolver`) 允许自定义扩展，例如：
		    - 自定义参数解析器（`HandlerMethodArgumentResolver`）支持新数据类型。
		    - 拦截器（`HandlerInterceptor`）在请求前后插入逻辑（如日志）。
		- **策略模式应用**：组件（如 `HandlerMapping`）支持多种实现，可灵活切换。
		- **产出结果**：开发者可深度定制流程，无需修改框架源码。
    
---
- 处理请求的核心组件

| 组件                        | 主要职责                             |
| ------------------------- | -------------------------------- |
| **DispatcherServlet**​    | 前端控制器，流程的总调度中心。                  |
| **HandlerMapping**​       | 根据请求URL找到对应的处理器。                 |
| **HandlerAdapter**​       | 使用统一的接口去实际执行不同类型的处理器。            |
| **HandlerInterceptor**​   | 拦截器，在请求处理的不同阶段切入通用逻辑（如权限检查）。     |
| **Controller**​           | 处理器，执行业务逻辑。                      |
| **ViewResolver**​         | 将逻辑视图名解析为具体的视图实现对象。              |
| **View**​                 | 负责将模型数据渲染成最终用户看到的格式（如HTML）。      |
| **HttpMessageConverter**​ | 处理请求/响应的数据转换（如JSON与Java对象的相互转换）。 |

- 请求处理流程
	- 核心思想：**前端控制器模式**，由`DispatcherServlet`作为总调度中心
	1. **请求接收与映射**
	    - **核心思路**：统一入口，动态查找。
	    - **实现机制**：
		    1. 所有匹配的请求首先由`DispatcherServlet`**接收**。
		    2. 它**咨询**一个或多个`HandlerMapping`组件（如`RequestMappingHandlerMapping`），根据请求的URL、HTTP方法等信息，**找到**能够处理该请求的**控制器方法。**
	    - **产出结果**：`HandlerMapping`返回一个`HandlerExecutionChain`对象，该对象封装了
		    - 目标**处理器**（Handler）
		    - 适用于该请求的所有**拦截器**（`HandlerInterceptor`）。
	2. **拦截预处理与适配调用**
	    - **核心思路**：横切关注点分离，统一调用接口。
	    - **实现机制**：
	        - **执行拦截 (`preHandle`)**：在调用业务逻辑之前，`DispatcherServlet`会执行`HandlerExecutionChain`中所有拦截器的`preHandle`方法
	        - **获取适配器 (`HandlerAdapter`)**：
		        - 需求背景：处理器形式多样，需要一個统一的接口来调用它们
			        -  `@Controller`
			        - `HttpRequestHandler`
			        - `Servlet`
		        - 解决措施：通过适配器`HandlerAdapter`来实际执行它。
			        - 如`RequestMappingHandlerAdapter`：用于适配 `@RequestMapping` 注解的方法
			        - 适配器的核心职责：
				        - **参数解析**：根据方法签名，使用各种 `HandlerMethodArgumentResolver` 来解析方法的参数
					        - 如解析`@RequestParam`, `@RequestBody`等注解
				        - **调用方法**：通过反射调用控制器方法
				        - **返回值处理**：使用方法返回值，使用各种 `HandlerMethodReturnValueHandler` 处理返回值
					        - 如 `@ResponseBody`, `ModelAndView`
		        - 核心思想：适配器模式
			        - 屏蔽了不同处理器的调用细节。
	    - **作用**：
		    - 确保公共逻辑（拦截器）在业务代码前后执行
		    - 并提供统一的处理器调用方式。
	3. **业务处理与视图渲染（关键分支）**
	    - **核心思路**：参数解析、业务执行、结果适配。
	    - **流程**：
	        1. `HandlerAdapter`会负责复杂的**参数绑定**（如解析`@RequestParam`, `@RequestBody`等注解）和**方法调用**。
	        2. 控制器方法执行完毕后，返回一个结果。此时，流程出现关键分支：
	            - **返回视图**：
		            - 返回类型：视图名（如String）或`ModelAndView`对象，
		            - 措施：
			            1. `DispatcherServlet`调用`ViewResolver`**将逻辑视图名解析为具体的`View`对象**（如JSP, Thymeleaf模板）
			            2. 由`View`对象将模型数据**渲染**成最终的HTML页面。
	            - **直接返回数据**：
		            - 前提：方法标记了`@ResponseBody`或控制器是`@RestController`
		            - 措施：
			            1. 结果会通过`HttpMessageConverter`（如`MappingJackson2HttpMessageConverter`）直接**序列化**为JSON/XML等格式
			            2. **写入响应体**，跳过视图解析步骤。
	4. **拦截后处理与资源清理**
	    - **核心思路**：确保后续处理与资源释放。
	    - **实现机制**：
	        - 执行`postHandle`方法：之后，拦截器的`postHandle`方法会被**逆序执行**，此时还可以对模型数据进行最后修改。
	        - 执行`afterCompletion`方法：最终，无论请求处理成功与否，拦截器的`afterCompletion`方法都会被调用，非常适合进行资源清理等工作。
	    - **作用**：完成请求处理的收尾工作。
	5. 异常处理
		- 异常传播与捕获流程
			1. **异常抛出**：在请求处理任何阶段（映射、拦截、业务执行等）抛出的异常
			2. **DispatcherServlet捕获**：`DispatcherServlet`的`doDispatch`方法捕获异常
			3. **遍历异常解析器**：调用`processHandlerException`遍历已注册的`HandlerExceptionResolver`
			4. **异常解决优先级**：
				1. **局部异常处理 (`@ExceptionHandler`)**：最内层、优先级最高的处理方式
				    - 定义方法：在单个`Controller`内部使用`@ExceptionHandler`注解一个方法
				    - 作用范围：该控制器中特定类型的异常。
				    - 使用场景：它非常适合处理某个控制器独有的异常。
				2. **全局异常处理 (`@ControllerAdvice`或 `@RestControllerAdvice`)** ：企业应用中最常用、最推荐的方式
				    
				    - 定义方式：定义一个被`@ControllerAdvice`注解的类，
				    - 作用范围：整个应用中所有控制器抛出的异常
					    - 匹配规则：系统会优先匹配异常类型最精确的处理方法。例如，同时存在处理`IOException`和`Exception`的方法，当抛出`IOException`时，会调用前者。
					- 好处：它确保了异常处理逻辑的统一性。
					    - 统一响应体：将异常信息转换为定义好的标准错误响应体（如`Result`对象）并返回，从而保证API响应格式的一致性。
				3. **兜底处理 (`BasicErrorController`)**
				    如果上述两层都没有处理异常（例如，未定义的404错误或未被前述处理器捕获的未知异常），Servlet容器（如Tomcat）会捕获该异常并将请求转发到 `/error`路径。这个请求由Spring Boot自动配置的 `BasicErrorController`处理。
				    - **内容协商**：`BasicErrorController`的一个智能之处在于它能根据客户端请求的`Accept`头信息，自动返回不同格式的错误信息。对浏览器请求（`Accept: text/html`）通常返回一个HTML错误页面；对API调用（如`Accept: application/json`），则返回JSON数据。这种机制称为“内容协商”。
				    


----
**异常处理构建**

在实际项目中，为了构建健壮且易维护的应用，通常会结合以下实践：

- 定义业务异常类：
	- 需求背景：
		- 在业务逻辑中，我们常常需要抛出一些具有特定含义的异常，例如“用户不存在”、“订单状态异常”等。
	- 解决措施：使用自定义的、含义明确的业务异常类，而非通用的`RuntimeException`
		- 可以极大地提升代码的**可读性**和异常信息的**丰富性**。
	- 实现机制
	    - 继承关系：继承 `RuntimeException`，属于非检查型异常
		    - 作用：在业务代码中抛出时无需强制捕获，保持代码简洁
		- 错误码（code）：
			- 作用：用于唯一标识一类错误，便于前端或调用方进行程序化判断
			- 实现方式：通过枚举类实现
		- 错误信息（message）：用于描述错误的详细信息，通常需要展示给用户或开发者
	- 使用方式：在业务层或控制层，通过 `throw new BusinessException(ErrorCode.USER_NOT_FOUND)`的方式抛出异常
- 定义全局异常处理器
	- 需求背景：如果在每个Controller方法中都使用`try-catch`来处理异常，会导致大量重复代码，且不易维护。
	- 解决措施：一个集中式的解决方案来统一捕获和处理所有控制器抛出的异常。
	- 实现方式：
		- 定义方式：
			- 利用 `@RestControllerAdvice`（或 `@ControllerAdvice`）注解来定义一个全局异常处理器。
			- 利用@ExceptionHandler指定一个方法是捕获和处理哪类自定义的业务异常
			- 利用@ResponseStatus指定返回的状态码
		- 拦截范围：整个应用程序中控制器（Controller）层抛出的异常
	- 作用：
		- 保证返回的是一个统一的接口规范类
		- 避免了敏感信息（如堆栈跟踪）直接暴露给用户
		  
- 利用Spring Boot的配置：
	- 需求背景：
		- 即使在有全局异常处理器的情况下，一些底层错误（如404 Not Found）可能先被Spring Boot的默认错误处理机制拦截。
		- 有敏感的系统内部信息（如异常堆栈）泄露给外部用户的风险
	- 解决措施：通过 `server.error`系列配置项，允许我们对其内置的`BasicErrorController`的行为进行精细化控制
		- 实例：
			```yaml
			# application.yml
			server:
			  error:
			    include-stacktrace: never # 永远不包含堆栈跟踪信息
			    # include-stacktrace: on_trace_param # 仅在请求带有trace参数时包含（常用于开发测试）
			    # include-stacktrace: always # 总是包含（不推荐用于生产）
			    include-exception: false   # 不包含异常类名
			    include-message: always    # 包含错误消息
			    include-binding-errors: always
			    path: /error              # 内置错误处理器的路径，通常无需修改		  
			```
    
- 自定义错误页面：
	- 需求背景：
		- 传统的服务端渲染应用，当发生404、500等错误时，直接展示一个JSON响应**并不友好**。我们需要展示一个符合产品风格的自定义HTML页面来提升**用户体验**。
	- 解决措施：Spring Boot支持通过约定大于配置的方式，自动识别并应用放置在特定目录下的自定义错误页面
	- 实现方式：
    1. **确定页面位置与命名**：
	    - 如果项目使用了Thymeleaf、FreeMarker等模板引擎，请在 `src/main/resources/templates/error/`目录下创建错误页面。
	    - 如果项目是纯静态页面，请在 `src/main/resources/static/error/`目录下创建错误页面。
	    - 页面的命名规则为：`状态码.html`（如`404.html`）或`状态码系列.html`（如`5xx.html`）。
	2. **创建错误页面**：
	    例如，创建一个友好的`404.html`页面：
		```
		<!DOCTYPE html>
		<html lang="zh-CN">
		<head>
		    <meta charset="UTF-8">
		    <title>页面未找到 - 404</title>
		    <style>/* 在这里添加你的CSS样式 */</style>
		</head>
		<body>
		    <div class="error-container">
		        <h1>404</h1>
		        <p>抱歉，您访问的页面不存在。</p>
		        <a href="/">返回首页</a>
		    </div>
		</body>
		</html>
		```
