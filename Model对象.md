### 存入数据
```java
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class ForwardController {

    @GetMapping("/source")
    public String sourcePage(Model model) { // Spring 会自动注入 Model 对象
        // 添加数据到 Model，相当于 request.setAttribute("message", "Hello from source!")
        model.addAttribute("message", "Hello from source!");
        model.addAttribute("user", new User("Alice"));

        // 转发到 /target 这个URL（可以是控制器映射，也可以是视图）
        return "forward:/target";
    }
```

### 获取数据
#### 1.使用 Model对象 参数
```
    @GetMapping("/target")
    public String targetPage(Model model) {
        // 在这里可以直接获取从 sourcePage 方法中传入的数据
        String message = (String) model.getAttribute("message");
        User user = (User) model.getAttribute("user");
        // 也可以添加新的数据
        model.addAttribute("newData", "Data added in target");
        // 最终渲染 /WEB-INF/views/result.jsp (或Thymeleaf等模板)
        return "result";
    }
}
```

#### 2.使用方法参数的注解`@ModelAttribute`
`@ModelAttribute`可以注解在方法参数上，用于从 `Model`中提取数据，非常方便。

```java
@GetMapping("/target")
public String targetPage(@ModelAttribute("message") String message, 
                         @ModelAttribute("user") User user, 
                         Model model) {
    // Spring 会自动从 Model 中取出 key 为 "message" 和 "user" 的值并注入到参数中
    System.out.println("Message: " + message);
    System.out.println("User: " + user.getName());
    
    return "result";
}
```