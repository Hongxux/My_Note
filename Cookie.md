：
![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/hy-20250914141649_e0d1c67c63455c2562f71481d7fc7a06-46-0.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1757856268%3B2073216268&q-key-time=1757856268%3B2073216268&q-header-list=host&q-url-param-list=&q-signature=adeef29884bd1d7c4cae6aa3f466266e7354a2ad)

![[Pasted image 20250914213156.png]]
**Cookie本质**：Cookie 是一小段文本信息，由服务器发送给浏览器（客户端），并由浏览器存储。之后，浏览器在每次向同一服务器发起请求时，都会自动携带上这些 Cookie 信息。

​**Cookie工作流程**​：

1. 客户端首次请求服务器
    
2. 服务器通过`Set-Cookie`首部字段返回标识信息
	- 这个 `Set-Cookie`头包含了服务器想要浏览器存储的键值对信息（例如一个会话 ID Session ID）和一些属性指令。
```
		HTTP/2.0 200 OK
		Content-Type: text/html
		Set-Cookie: session_id=abc123xyz; Path=/; Secure;           HttpOnly; SameSite=Lax
```
    
3. 客户端保存Cookie并在后续请求中自动携带
    - 浏览器收到服务器的响应后，会解析响应头中的 `Set-Cookie`字段。
	- 浏览器会**按照指令的要求**​（如有效期 `Expires`或 `Max-Age`），将 Cookie 的键值对及其属性**本地存储**在用户电脑上（通常在一个特定的 cookie 文件中）。
	- 此后，只要 Cookie 没有过期，且符合路径（`Path`）和域名（`Domain`）等规则，浏览器就会在请求该网站时使用它。
4. 服务器通过Cookie识别用户状态

	- 服务器收到请求后，会读取 `Cookie`请求头。
	- 它通过解析 `Cookie`头中的值（例如 `session_id=abc123xyz`），就可以识别出这是哪个用户发起的请求。
	- 服务器根据这个身份信息，从数据库或缓存中获取该用户的个人数据（如用户名、偏好设置等），然后生成一个个性化的页面作为响应返回给浏览器。
	- 这样，用户就看到了自己已登录的状态和个人信息，完成了“保持登录”等一系列需要状态管理的功能。
```
GET /my_account HTTP/2.0
Host: www.example.com
Cookie: session_id=abc123xyz; user_language=en-US
```
