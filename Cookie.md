

![[Pasted image 20250914213156.png]]
**Cookie本质**：Cookie 是一小段文本信息
- 来源：由服务器发送给浏览器（客户端）
	- 提供`Set-Cookie`首部字段返回标识信息
		- `Set-Cookie`的内容：
			- 想要浏览器存储的键值对信息
			- 属性指令（有效期等等）
- 存储地方：浏览器本地
- 使用方式：每次向同一服务器发起请求时，自动携带上这些 Cookie 信息
	- 携带的前提：
		- 符合路径和域名等规则
		- Cookie未过期
	- 服务器识别：
		- 通过`Cookie`头中的值（例如 `session_id=abc123xyz`）识别出是哪个用户

​来自服务器
```
HTTP/2.0 200 OK
Content-Type: text/html
Set-Cookie: session_id=abc123xyz; Path=/; Secure;           HttpOnly; SameSite=Lax
```
    
来自客户端
```
GET /my_account HTTP/2.0
Host: www.example.com
Cookie: session_id=abc123xyz; user_language=en-US
```
