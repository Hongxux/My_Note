---
aliases:
  - HTTP响应、HTTP请求
---


## 一、HTTP报文基本结构

HTTP协议通信的核心是通过请求报文和响应报文实现的，这两种报文具有相似的结构，都包含以下三部分：
![[Pasted image 20250914202739.png]]![[Pasted image 20250914202758.png]]
- ​**起始行**​：请求报文为请求行，响应报文为状态行
    
- ​**首部字段**​：包含多个键值对，描述报文属性和特征
    
- ​**报文主体**​：实际传输的数据内容（可选）
    

## 二、[[HTTP请求报文]]构成

    

## 三、[[HTTP响应报文]]构成


## 四、HTTP无状态特性与[[Cookie]]管理

HTTP协议本身是无状态的，不会保留之前的请求信息。为了维持状态，使用Cookie技术
## 五、[[URI访问形式]]与HTTP方法

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
    

