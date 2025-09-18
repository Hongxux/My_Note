# HTTP报文详解：构成、传输与交互机制

## 一、HTTP报文基本结构

HTTP协议通信的核心是通过请求报文和响应报文实现的，这两种报文具有相似的结构，都包含以下三部分：
![[Pasted image 20250914202739.png]]![[Pasted image 20250914202758.png]]
- ​**起始行**​：请求报文为请求行，响应报文为状态行
    
- ​**首部字段**​：包含多个键值对，描述报文属性和特征
    
- ​**报文主体**​：实际传输的数据内容（可选）
    

## 二、请求报文构成
![[Pasted image 20250914202919.png]]请求报文首部分为请求行和HTTP首部字段
### 2.1 请求行

请求报文的第一行为请求行，包含三个基本要素：

```
GET /index.html HTTP/1.1
```

- ​**方法**​：指定操作类型（GET、POST等）
    
- ​**请求URI**​：标识目标资源
    
- ​**HTTP版本**​：使用的协议版本
    

### 2.2 请求首部字段

![[Pasted image 20250914202758.png]]

请求首部字段分为四类：

- **[[通用首部字段]]（General Header Fields）：** 请求报文和响应报文两方都会使用的首部。 
- **[[请求首部字段]]（Request Header Fields）**: 从客户端向服务器端发送请求报文时使用的首部。补充了请求的附加 内容、客户端信息、响应内容相关优先级等信息。
- **[[实体首部字段]]（Entity Header Fields）:** 针对请求报文和响应报文的实体部分使用的首部。补充了资源内容更新时间等与实体有关的信息。
    

## 三、响应报文构成

### 3.1 状态行

响应报文的第一行为状态行，包含三个要素：

```
HTTP/1.1 200 OK
```

- ​**HTTP版本**​：协议版本
    
- ​ **[[返回结果的HTTP状态|状态码]]** ​ ：3位数字，表示请求处理结果
    
- ​**原因短语**​：状态码的文本描述
    

### 3.2 响应首部字段

![[Pasted image 20250914203015.png]]
- **[[通用首部字段]]（General Header Fields）：** 请求报文和响应报文两方都会使用的首部。 
- **[[响应首部字段]]（Response Header Fields）:** 从服务器端向客户端返回响应报文时使用的首部。补充了响应的附加 内容，也会要求客户端附加额外的内容信息。 
- **[[实体首部字段]]（Entity Header Fields）:** 针对请求报文和响应报文的实体部分使用的首部。补充了资源内容更新时间等与实体有关的信息。

## 四、HTTP无状态特性与Cookie管理

HTTP协议本身是无状态的，不会保留之前的请求信息。为了维持状态，使用Cookie技术：
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

## 五、URI访问形式与HTTP方法

### 5.1 URI的两种访问形式

1. ​**访问特定资源**​
    
    ```
    GET /index.html HTTP/1.1
    ```
    
    直接获取服务器指定路径的资源
    
2. ​**对服务器本身发起请求**​
    
    ```
    OPTIONS * HTTP/1.1
    ```
    
    查询服务器支持的方法或测试服务器功能
    
### 5.2URI和URL的区别：
URI（Uniform Resource Identifier）即 **[[统一资源标识符]]**，是通过字符串格式唯一标识互联网资源的方式（RFC 2396）。

### 5.3 主要HTTP方法

|方法|作用|幂等性|示例场景|
|---|---|---|---|
|`GET`|获取资源|是|访问网页|
|`POST`|提交数据|否|用户登录|
|`PUT`|更新资源|是|上传文件|
|`DELETE`|删除资源|是|删除数据|
|`HEAD`|获取首部|是|检查更新|
|`OPTIONS`|查询支持方法|是|预检请求|

![[Pasted image 20250914203031.png]]

## 六、连接管理与性能优化

### 6.1 持久连接（Keep-Alive）

早期HTTP每次请求都需要建立和断开TCP连接，效率低下。持久连接允许单个TCP连接处理多个HTTP请求。

​**性能对比**​：

![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/hy-20250914141649_e0d1c67c63455c2562f71481d7fc7a06-42-0.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1757856287%3B2073216287&q-key-time=1757856287%3B2073216287&q-header-list=host&q-url-param-list=&q-signature=413bd1c3507889413e73ea5921363d77dd8a2f8d)

### 6.2 管线化（Pipelining）
![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/hy-20250914141649_e0d1c67c63455c2562f71481d7fc7a06-45-1.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1757856291%3B2073216291&q-key-time=1757856291%3B2073216291&q-header-list=host&q-url-param-list=&q-signature=90184ea216709a2522a2c37c2c9d55b03fde0e43)
**管线化（Pipelining）代表了一种旨在提升HTTP/1.1性能的优化技术，它允许在等待响应之前连续发送多个请求。​**​ 虽然由于其自身的设计缺陷（如队头阻塞）而未能成功，但它为现代网络协议的发展铺平了道路。其核心思想被 ​**HTTP/2的多路复用**​ 完美实现和超越，成为了当今网络高速传输的基石。
## 七、内容处理机制

### 7.1 内容编码

为减少传输数据量，HTTP支持内容编码压缩：

- ​**常见算法**​：gzip、deflate、br
    ![[Pasted image 20250914213105.png]]
- ​**流程**​：服务器压缩 → 传输 → 客户端解压
    ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/hy-20250914141649_e0d1c67c63455c2562f71481d7fc7a06-52-1.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1757856295%3B2073216295&q-key-time=1757856295%3B2073216295&q-header-list=host&q-url-param-list=&q-signature=647f4fa572b3fe290e63cc43bd76a0392c97aa18)

### 7.2 分块传输编码
[[协议栈中的四种“分块”​|HTTP分块编码、TCP分块，IP分块编码，以及MAC模块分块编码有什么区别，工作原理分别是什么]]
用于动态生成内容，将数据分成多个块依次传输，便于服务器边生成边发送。
![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/hy-20250914141649_e0d1c67c63455c2562f71481d7fc7a06-53-0.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1757856299%3B2073216299&q-key-time=1757856299%3B2073216299&q-header-list=host&q-url-param-list=&q-signature=83d647d52f87247eb89c6f55e0bb0bd76f9220bc)
### 7.3 多部分对象集合

单报文中传输多种类型数据，常见于：

- `multipart/form-data`：文件上传
    
- `multipart/byteranges`：多范围响应
    

### 7.4 范围请求

支持请求资源的特定部分，用于断点续传、视频跳播等场景：

- 请求头：`Range: bytes=500-1000`
    
- 响应头：`Content-Range: bytes 500-1000/1500`
    

## 八、内容协商机制

服务器根据客户端能力提供最合适资源，三种协商方式：

1. ​**服务器驱动协商**​：服务器根据请求头（如Accept-Language）决定返回资源
    ![](https://hunyuan-plugin-private-1258344706.cos.ap-nanjing.myqcloud.com/pdf_youtu/img/hy-20250914141649_e0d1c67c63455c2562f71481d7fc7a06-57-4.png?q-sign-algorithm=sha1&q-ak=AKID372nLgqocp7HZjfQzNcyGOMTN3Xp6FEA&q-sign-time=1757856313%3B2073216313&q-key-time=1757856313%3B2073216313&q-header-list=host&q-url-param-list=&q-signature=0e097f6f12b80a6c4967a42eb3b5c778c38d3dbe)
2. ​**客户端驱动协商**​：服务器返回可选列表，用户手动选择
    
3. ​**透明协商**​：中间设备（如代理）负责协商
    

