---
aliases:
  - DispatcherServlet
---
好的，我们从源码的角度深入剖析 Spring MVC 的请求处理流程。整个过程的核心是 **`DispatcherServlet`**，它作为前端控制器，是整个 MVC 框架的中枢神经，协调所有组件完成请求处理。

为了直观地把握全局，下图描绘了 `DispatcherServlet`的 `doDispatch`方法处理一个 HTTP 请求的完整工作流程与核心组件的协作关系：

### 二、 源码深度解析：`doDispatch`方法

`doDispatch`是 `DispatcherServlet`类（位于 `org.springframework.web.servlet`包）中最核心的方法。它处理所有到达的请求。

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    ModelAndView mv = null;
    Exception dispatchException = null;

    try {
        // 1. 预处理请求（如文件上传）
        processedRequest = checkMultipart(request);
        response = checkMultipart(response);

        // 2. 为当前请求确定一个处理程序执行链（Handler + Interceptors）
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) {
            noHandlerFound(processedRequest, response); // 404处理
            return;
        }

        // 3. 获取支持该处理程序的适配器
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // 4. 处理最后修改的HTTP头（缓存相关）
        String method = request.getMethod();
        boolean isGet = "GET".equals(method);
        if (isGet || "HEAD".equals(method)) {
            long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
            if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                return;
            }
        }

        // 5. 【关键】按顺序应用拦截器的 preHandle 方法
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return; // 如果任一 preHandle 返回 false，中断流程
        }

        // 6. 【核心】实际调用控制器方法，由 HandlerAdapter 处理
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        // 7. 如果默认需要设置视图名，则设置
        applyDefaultViewName(processedRequest, mv);

        // 8. 【关键】按逆序应用拦截器的 postHandle 方法
        mappedHandler.applyPostHandle(processedRequest, response, mv);

    } catch (Exception ex) {
        dispatchException = ex; // 捕获处理过程中抛出的异常
    } catch (Throwable err) {
        dispatchException = new NestedServletException("Handler dispatch failed", err);
    }

    // 9. 处理分发结果（渲染视图、处理异常）并触发 afterCompletion
    processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
}
```

### 三、 关键步骤详解

#### 步骤 2：`getHandler()`- 查找处理器执行链

这个方法负责根据当前请求的 URL 等信息，找到对应的控制器（Handler）和配置的拦截器链。

```
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        // 遍历所有已注册的 HandlerMapping 组件
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler; // 找到即返回
            }
        }
    }
    return null; // 未找到
}
```

- **`HandlerMapping`**：策略接口，实现类（如 `RequestMappingHandlerMapping`）维护了 URL 到控制器方法的映射关系。
    
- **`HandlerExecutionChain`**：不仅包含目标 `Handler`（你的 `@Controller`中的方法），还包含了所有配置应用于该请求的 `HandlerInterceptor`。
    

#### 步骤 3：`getHandlerAdapter()`- 获取处理器适配器

Spring MVC 设计了**适配器模式**来支持不同类型的处理器（如 `@Controller`、`Controller`接口、`HttpRequestHandler`）。

```
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    if (this.handlerAdapters != null) {
        // 遍历所有已注册的 HandlerAdapter
        for (HandlerAdapter adapter : this.handlerAdapters) {
            if (adapter.supports(handler)) { // 判断该适配器是否支持当前处理器
                return adapter;
            }
        }
    }
    throw new ServletException("No adapter for handler [" + handler + "]");
}
```

- **`HandlerAdapter`**：策略接口。例如，`RequestMappingHandlerAdapter`负责处理我们最常用的 `@RequestMapping`注解的方法。
    

#### 步骤 6：`ha.handle()`- 适配器执行控制器方法

这是真正调用你的控制器方法的地方。以 `RequestMappingHandlerAdapter`为例，其 `handle()`方法最终会调用 `invokeHandlerMethod()`。

```
// 在 RequestMappingHandlerAdapter 中
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

    // 1. 创建方法参数解析器，将HTTP请求数据解析为控制器方法的参数
    // （如 @RequestParam, @RequestBody, @PathVariable 等注解的解析都在这里完成）
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
    ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

    // 2. 创建方法调用容器
    ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
    invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers); // 设置参数解析器
    invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers); // 设置返回值处理器

    // ... 更多设置 ...

    // 3. 实际调用控制器方法并处理返回值
    invocableMethod.invokeAndHandle(webRequest, mavContainer);

    // 4. 返回 ModelAndView
    return getModelAndView(mavContainer, modelFactory, webRequest);
}
```

这一步是**依赖注入**和**注解驱动**的魔法发生的地方。

#### 步骤 9：`processDispatchResult()`- 处理最终结果

这个方法负责视图渲染、异常处理，并最终触发拦截器的 `afterCompletion`。

```
private void processDispatchResult(HttpServletRequest request, HttpServletResponse response,
        @Nullable HandlerExecutionChain mappedHandler, @Nullable ModelAndView mv,
        @Nullable Exception exception) throws Exception {

    // 1. 处理异常（如果有的话）
    if (exception != null) {
        if (exception instanceof ModelAndViewDefiningException) {
            // ... 处理异常 ...
        } else {
            // 触发异常解析器（如 @ExceptionHandler）
            Object handler = (mappedHandler != null ? mappedHandler.getHandler() : null);
            mv = processHandlerException(request, response, handler, exception);
        }
    }

    // 2. 渲染视图（如果存在视图）
    if (mv != null && !mv.wasCleared()) {
        render(mv, request, response);
    }

    // 3. 无论成功与否，最终触发拦截器的 afterCompletion 回调
    if (mappedHandler != null) {
        mappedHandler.triggerAfterCompletion(request, response, null);
    }
}
```

**`render()`方法会调用 `ViewResolver`来解析视图名：**

```
protected void render(ModelAndView mv, HttpServletRequest request, HttpServletResponse response) throws Exception {
    // 解析逻辑视图名（如 "user/profile"）到具体的 View 对象（如 JstlView）
    View view;
    String viewName = mv.getViewName();
    if (viewName != null) {
        view = resolveViewName(viewName, mv.getModelInternal(), locale, request);
    } else {
        view = mv.getView();
    }
    // 调用 View 对象的 render 方法，将模型数据渲染到响应中
    view.render(mv.getModelInternal(), request, response);
}
```

### 四、 总结：设计哲学

通过源码分析，我们可以看到 Spring MVC 的设计非常精妙：

1. **前端控制器模式**：`DispatcherServlet`是唯一入口，统一调度，职责清晰。
    
2. **职责链模式**：通过 `HandlerExecutionChain`将处理器和多个拦截器组合，灵活添加公共逻辑。
    
3. **适配器模式**：`HandlerAdapter`屏蔽了不同处理器的实现差异，极大地提高了框架的扩展性。
    
4. **策略模式**：几乎所有核心组件（`HandlerMapping`, `HandlerAdapter`, `ViewResolver`）都是接口，允许用户自定义实现。
    
5. **模板方法模式**：`doDispatch`定义了请求处理的骨架，将具体步骤延迟到子组件中实现。
    

**简单来说，Spring MVC 的执行流程是一个高度模块化、可扩展的“流水线作业”**。`DispatcherServlet`是总指挥，它自己不干活，而是知道在哪个环节该调用哪个专家（组件）来处理，最终将一次 HTTP 请求优雅地转换成一个完整的响应。理解这个流程，对深入掌握和高效使用 Spring MVC 至关重要。