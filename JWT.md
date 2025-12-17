- 需求背景：传统Session认证的痛点
	- 会话数据存储在服务器内存，给服务器带来负担， 在集群部署时需要额外机制（如Session共享）来支持横向扩展
		- 比如使用Session复制，占用存储空间
		- 比如使用集中式Seesion存储（Redis）,引入外部依赖和网络开销的代价
		- 比如[[粘连会话]]，有单点故障风险
	- 主要基于Cookie，在应对多种前端设备（如移动端）和跨域（CORS）场景时不够灵活
	- ​容易受到跨站请求伪造（CSRF）**​ 攻击
- 解决措施：JWT
	- **无状态**：服务器不存储会话状态，易于水平扩展
	- **跨域友好**：通常通过HTTP头部（如`Authorization: Bearer <token>`）传输，天然支持跨域API调用和多种客户端
	- **抗CSRF**：不依赖Cookie，有效避免CSRF攻击。令牌本身有签名，可防篡改
- 组成成分：
	- Header（头部）：通常是一个JSON对象，经过Base64URL编码后形成
		- 令牌类型（`typ`）
		- 签名算法（`alg`）
			- 问题：`alg:none`攻击、
			- 含义：当攻击者将`alg`从`RS256`（非对称算法）修改为`none`后，服务器可能会跳过签名验证步骤，直接信任Payload中的内容
			- 解决措施：明确指定允许使用的算法列表`Jwts.parser().setSigningKey(publicKey).setAllowedAlgorithms(["RS256"]).build().parseClaimsJws(token)`
	- Payload（负载）：关于实体（如用户）和其他数据的陈述。
		- 注册声明：预定义但不强制使用的标准字段，如签发人（`iss`）、过期时间（`exp`）、主题（`sub`）
		- 公共声明：可自定义的字段
		- 私有声明：通信双方协商定义的自定义字段
	- Signature（签名）：**防止令牌内容在传输过程中被篡改**
		- 计算方法：通过对编码后的Header、编码后的Payload以及一个只有服务器知道的密钥（`secret`）使用Header中指定的算法（如HS256）计算得出
	- 一个完整的JWT看起来是这样的（由点号.分隔的三部分）：`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c`
- 企业要解决的问题：、
	- **令牌废止**：由于JWT是无状态的，服务器一旦签发，在有效期内无法直接使其失效。
	- **数据过期**：负载中的数据在令牌过期前无法更新
	- [[令牌续期]]
	- [[密匙管理]]
- 注意：
	- JWT的Header和Payload部分仅是Base64URL编码，**并非加密**。因此，**切勿在JWT中放置密码等敏感信息**
		- 解决措施：[[令牌黑名单]]
		

-  JWT的工作流程
	1. 用户登录：用户使用凭证（如用户名和密码）向认证服务端发起登录请求。
	2. 服务器验证并生成JWT：服务器验证凭证有效后，会生成一个JWT。
		- JWT的负载：通常包含用户标识（如用户ID）、角色权限等信息，
		- JWT的过期时间：设置一个较短的过期时间（`exp`）。
		- JWT的签名：服务器使用一个保密密钥（用于HMAC算法）或一对公私钥（用于RSA算法）为这个JWT生成签名。
	3. 返回JWT给客户端：服务器将生成的JWT返回给客户端。客户端通常会将其存储在本地，例如浏览器的`localStorage`、`sessionStorage`或Cookie中。
	4. 客户端携带JWT发起请求：客户端在后续访问受保护资源或API时，需要在HTTP请求的`Authorization`头部中携带此JWT，通常使用`Bearer`模式：`Authorization: Bearer <your-token>`。
	5. 服务器验证JWT：服务器收到请求后，会从`Authorization`头部提取JWT，然后进行验证。验证包括检查签名是否有效、Token是否已过期（检查`exp`声明）等。
	6. 返回响应：如果JWT验证通过，服务器则处理请求并返回请求的数据。如果验证失败（如签名无效或已过期），服务器会返回错误状态码（如401 Unauthorized）。
- 使用方式：
	 1. 添加依赖
	```java
	<dependency>
	    <groupId>com.auth0</groupI>
	    <artifactId>java-jwt</artifactId>
	    <version>4.4.0</version> <!-- 请使用当前最新稳定版本 -->
	</dependency>
	```
	2. 生成 JWT：在用户登录成功后，使用JWT库生成Token。
	```java
	import com.auth0.jwt.JWT;
	import com.auth0.jwt.algorithms.Algorithm;
	import java.util.Date;
	
	public class JwtUtil {
	    // 一个保密的密钥，非常重要，必须妥善保管且足够复杂
	    private static final String SECRET_KEY = "your-super-secret-key-with-sufficient-length";
	    // 设置Token有效时间，例如1小时
	    private static final long EXPIRATION_TIME = 3600_000; // 毫秒
	
	    public static String generateToken(String userId, String username) {
	        Algorithm algorithm = Algorithm.HMAC256(SECRET_KEY); // 使用HS256算法
	        return JWT.create()
	                .withIssuer("your-issuer") // 签发者（可选）
	                .withSubject(userId)       // 主题（通常放用户ID）
	                .withClaim("username", username) // 自定义声明
	                .withIssuedAt(new Date())   // 签发时间
	                .withExpiresAt(new Date(System.currentTimeMillis() + EXPIRATION_TIME)) // 过期时间
	                .sign(algorithm); // 签名
	    }
	}
	```
	- **使用命令行工具（推荐，跨平台）** 生成SECRET_KEY
	在终端中执行以下命令，可以快速生成一个符合强度要求的Base64编码密钥：
	```
	# 生成一个32字节（256位）的随机密钥，并以Base64编码输出
	openssl rand -base64 32
	# 示例输出：aBcDeFgHiJkLmNoPqRsTuVwXyZ0123456789+AB/CDE=
	```
	- [[JWT参数]]
	###### 3. 验证并解析 JWT
	编写一个拦截器（Interceptor）或过滤器（Filter）来验证客户端请求中携带的JWT。
	```java
	import com.auth0.jwt.JWT;
	import com.auth0.jwt.algorithms.Algorithm;
	import com.auth0.jwt.exceptions.JWTVerificationException;
	import com.auth0.jwt.interfaces.DecodedJWT;
	
	public class JwtUtil {
	    // ... generateToken 方法同上 ...
	
	    public static DecodedJWT verifyToken(String token) throws JWTVerificationException {
	        // 使用相同的密钥和算法创建验证器
	        Algorithm algorithm = Algorithm.HMAC256(SECRET_KEY);
	        com.auth0.jwt.JWTVerifier verifier = JWT.require(algorithm)
	                .withIssuer("your-issuer") // 验证签发者，如果生成时设置了的话
	                .build();
	        // 验证Token，如果无效（如签名错误、过期）会抛出异常
	        return verifier.verify(token);
	    }
	
	    // 从已验证的Token中解析信息
	    public static String getUserIdFromToken(String token) {
	        DecodedJWT decodedJWT = verifyToken(token);
	        return decodedJWT.getSubject(); // 获取Subject，即用户ID
	    }
	}
	```
	- [[verify()的作用]]
	在拦截器中应用验证：记得注册这个拦截器到需要认证的接口路径。
	```java
	@Component
	public class JwtInterceptor implements HandlerInterceptor {
	    @Override
	    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
	        String authHeader = request.getHeader("Authorization");
	        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
	            // 没有携带Token，返回未认证错误
	            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
	            return false;
	        }
	        String token = authHeader.substring(7); // 去掉 "Bearer " 前缀
	        try {
	            DecodedJWT decodedJWT = JwtUtil.verifyToken(token);
	            String userId = decodedJWT.getSubject();
	            // 可以将用户信息存入请求上下文（如ThreadLocal），方便后续业务使用
	            // UserContext.setCurrentUserId(userId);
	            return true; // 验证通过，放行
	        } catch (JWTVerificationException e) {
	            // Token无效（签名错误、过期等）
	            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
	            return false;
	        }
	    }
	}
	```

- 