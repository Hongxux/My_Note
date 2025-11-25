HTTPS协议本质上是HTTP协议和SSL/TLS 协议的组合![[Pasted image 20251123215813.png]]
- SSL 是 “Secure Sockets Layer” 的缩写，中文意思为“安全套接层”，而 TLS 则是标准化之后的 SSL。
- TLS提供的功能有三个：
	- 数据私密性：使用对称加密算法用于加密数据的传输
	- 数据完整性：发送的每个消息都使用 MAC（消息认证码） 进行完整性检查
	- 通信方身份的可靠性：使用公钥加密来验证通信方的身份



# TLS 1.2的完整握手
​
![[Pasted image 20251123221207.png]]


## 基于ECDHE的四次TLS握手协议

[图解 ECDHE 密钥交换算法-CSDN博客](https://blog.csdn.net/m0_50180963/article/details/113061162)

- 最终会话密匙（对称加密）的生成：由「客户端随机数 + 服务端随机数 + x（ECDHE 算法算出的共享密钥） 」三个材料生成的
	- 使用`ClientRandom`+ `ServerRandom`+ `ECDHE 算法算出的共享密钥`，通过[[伪随机函数（PRF）]]​**​ 生成48字节`MasterSecret`
		- [[ClientRandom与ServerRandom在SSL、TLS握手中的必要性|为什么要用ClientRandom和ServerRandom]]
-  [[证书]]确保公开密匙的真实性

