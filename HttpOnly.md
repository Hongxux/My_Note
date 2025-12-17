- 需求背景：窃取用户Cookie，实现**会话劫持**，冒充用户身份
	- 恶意脚本可轻易通过 `document.cookie`读取所有Cookie（包括会话ID）
- 解决措施：HttpOnly
	- 恶意脚本无法读取受保护的Cookie内容
- 工作模式：
	- **服务器端设置**：当服务器向用户的浏览器发送Cookie时，通过在HTTP响应头 `Set-Cookie`中添加 `HttpOnly`属性来启用此保护 。例如：`Set-Cookie: sessionid=abc123; HttpOnly; Secure; SameSite=Strict`
	- **浏览器端强制执行**：浏览器识别到这个标志后，就会严格限制对该Cookie的访问。
		- 它会允许该Cookie在每次向同一站点发起HTTP请求时**自动携带**
		- 但会**禁止客户端JavaScript通过任何方式（如 `document.cookie`API）读取其值**
- 设置方式：
	```java
	
	@RestController
	public class SecureCookieController {
	
	    @GetMapping("/set-cookie-v2")
	    public String setCookieWithResponseCookie(HttpServletResponse response) {
	        // 使用ResponseCookie构建器
	        ResponseCookie cookie = ResponseCookie.from("session_id", "encrypted_session_value")
	                .httpOnly(true)      // 禁止JavaScript访问
	                .secure(true)        // 仅通过HTTPS传输
	                .path("/")           // 应用路径
	                .maxAge(Duration.ofHours(1)) // 有效期1小时
	                .sameSite("Lax")    // 设置SameSite属性以防CSRF
	                .domain("yourdomain.com") // 可选：设置域名
	                .build();
	
	        // 通过响应头设置Cookie
	        response.setHeader(HttpHeaders.SET_COOKIE, cookie.toString());
	        return "Cookie已安全设置";
	    }
	}
	```