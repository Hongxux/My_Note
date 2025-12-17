


1. 结构解析与解码
	- `verify()`方法首先会检查令牌的整体结构是否为有效的三段式格式（Header.Payload.Signature），并用点号分隔。接着，它对 Header 和 Payload 部分进行 **Base64Url 解码**。如果令牌格式明显错误或无法解码，会抛出 `JWTDecodeException`。
 2. 签名验证 - 防篡改的核心
	 - 验证器（Verifier）会使用你在 `JWT.require()`方法中提供的**算法（如 HS256）和密钥（Secret）**，对解码后的 Header 和 Payload 重新计算一次签名。然后，将计算出的新签名与 JWT 自带的第三部分（Signature）进行比对。
		- **如果签名匹配**：证明令牌是完整的、可信的，因为它是由持有相同密钥的合法签发方创建的。
		- **如果签名不匹配**：说明令牌的 Header 或 Payload 可能在传输过程中被修改了。此时会立即抛出 `SignatureVerificationException`异常，验证失败。
3. 标准声明验证 - 检查有效期
	- **过期时间（`exp`）**：检查当前时间是否在令牌声明的过期时间之前。如果令牌已过期，会抛出 `TokenExpiredException`。
	- **生效时间（`nbf`）**：如果存在此声明，会检查当前时间是否已晚于令牌的生效时间。
	- **签发时间（`iat`）**：有时也会用于辅助验证。
	许多库支持设置一个“宽容时间”（leeway）来解决服务器之间微小的时间差问题。
4. 自定义声明验证 - 业务规则检查
	- 在构建 `JWTVerifier`时，你不仅可以使用 `.withIssuer()`、`.withAudience()`等方法来验证标准声明，还可以通过 `.withClaim("key", "value")`来验证自定义声明。
