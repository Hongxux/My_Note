第一个请求的 `request`和 `response`对象在重定向发生后就已经销毁了。因此，不能直接用 `Model`来传递数据。Spring 提供了 `RedirectAttributes`专门解决此问题。

`RedirectAttributes`是 `Model`的子接口，提供了两种在重定向中传递数据的机制：
- 传递**简单、非敏感信息**（如ID、状态码）且**希望保留在历史记录中**时，用 **`addAttribute()`**。
    
- 传递**敏感信息、复杂对象**或**不希望暴露在URL中**的信息时，总是使用 **`addFlashAttribute()`**。这是现代Web开发中
#### 1. `addAttribute()`- 通过 URL 查询参数传递

这种方式会将数据直接拼接到重定向的 URL 后面，成为查询字符串（如 `?name=value&name2=value2`）。


```
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@PostMapping("/submitForm")
public String handleFormSubmit(FormData formData, RedirectAttributes attributes) {
    // ... 处理表单逻辑
    
    // 方法1: 使用 addAttribute - 数据会暴露在URL中
    attributes.addAttribute("status", "success"); // URL 会变成 /result?status=success
    attributes.addAttribute("userId", savedUser.getId()); // /result?status=success&userId=123
    
    // 重定向到 /result
    return "redirect:/result";
}

@GetMapping("/result")
public String showResultPage(@RequestParam("status") String status, 
                            @RequestParam("userId") Long userId, 
                            Model model) {
    // 从URL参数中获取数据
    model.addAttribute("message", "Status: " + status);
    return "result";
}
```

**特点**：

- **数据暴露**：所有参数都会显示在浏览器地址栏。
    
- **类型限制**：只能传递简单类型（String, 数字等），不适合复杂对象。
    
- **长度限制**：URL 有长度限制，不宜传递过多数据。
    

#### 2. `addFlashAttribute()`- 通过 Session 临时传递（推荐）

这是更优雅、更常用的方式。`Flash`属性会**临时存储在 Session 中**，在重定向后的**下一个请求中立即可用**，随后**自动清除**。


```
@PostMapping("/submitOrder")
public String submitOrder(Order order, RedirectAttributes attributes) {
    Order savedOrder = orderService.save(order);
    
    // 方法2: 使用 addFlashAttribute - 数据不会暴露在URL中
    attributes.addFlashAttribute("flashMessage", "Order placed successfully!");
    attributes.addFlashAttribute("order", savedOrder); // 甚至可以传递复杂对象
    
    return "redirect:/order/confirmation"; // 重定向到确认页面
}

@GetMapping("/order/confirmation")
public String showConfirmationPage(Model model) {
    // Flash 属性会自动从 Session 中取出并添加到 Model 中。
    // 所以我们通常不需要特殊处理，直接在方法中使用即可。
    
    // 在视图中，可以直接通过 ${flashMessage} 和 ${order} 来访问
    return "confirmation";
}
```

**特点**：

- **安全**：数据不会暴露在 URL 中。
    
- **强大**：可以传递任意可序列化的对象。
    
- **自动管理**：Spring 会自动在下次请求后清除这些属性，避免塞满 Session。
    
- **是“重定向后获取数据”模式（Post-Redirect-Get, PRG）的理想选择**。