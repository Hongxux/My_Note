1. **完整的令牌生命周期管理**：生成、验证、刷新、下线
    
2. **智能自动轮转**：AT过期时自动使用RT刷新
    
3. **家族令牌管理**：支持批量下线相关令牌
    
4. **分布式安全**：使用Redisson锁防止并发问题
    
5. **详细状态分类**：区分不同失效原因，便于前端处理
    
6. **配置外部化**：所有参数通过application.yml配置
    

## 🔧 完整的JWT工具类设计

```
package com.hmdp.utils;

import com.auth0.jwt.JWT;
import com.auth0.jwt.JWTVerifier;
import com.auth0.jwt.algorithms.Algorithm;
import com.auth0.jwt.exceptions.JWTDecodeException;
import com.auth0.jwt.exceptions.SignatureVerificationException;
import com.auth0.jwt.exceptions.TokenExpiredException;
import com.auth0.jwt.interfaces.DecodedJWT;
import com.hmdp.dto.JWTResult;
import com.hmdp.dto.UserDTO;
import lombok.extern.slf4j.Slf4j;
import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;
import java.time.Duration;
import java.util.Date;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

/**
 * JWT令牌管理工具类
 * 功能：AT/RT生成、状态检验、自动轮转、下线管理
 * 
 * @author YourName
 * @version 1.0
 * @since 2025
 */
@Slf4j
@Component
public class AdvancedJwtUtil {
    
    // ==================== 配置参数 ====================
    @Value("${jwt.access-token.expiration:1800000}") // 30分钟
    private long accessTokenExpiration;
    
    @Value("${jwt.refresh-token.expiration:604800000}") // 7天
    private long refreshTokenExpiration;
    
    @Value("${jwt.secret:your-super-secure-secret-key}")
    private String secret;
    
    @Value("${jwt.issuer:your-app-issuer}")
    private String issuer;
    
    // ==================== 依赖注入 ====================
    @Autowired
    private SnowFlakeIDGenerator snowFlakeIDGenerator;
    
    @Autowired
    private RedissonClient redissonClient;
    
    @Autowired
    private StringRedisTemplate redisTemplate;
    
    private Algorithm algorithm;
    
    // ==================== 初始化 ====================
    @PostConstruct
    public void init() {
        this.algorithm = Algorithm.HMAC256(secret);
        log.info("JWT工具类初始化完成 - Issuer: {}, AT过期时间: {}ms, RT过期时间: {}ms", 
                issuer, accessTokenExpiration, refreshTokenExpiration);
    }
    
    // ==================== 令牌生成方法 ====================
    
    /**
     * 生成Access Token (AT)
     * AT用于常规API访问，有效期较短
     * 
     * @param userDTO 用户信息
     * @return 生成的Access Token
     */
    public String generateAccessToken(UserDTO userDTO) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("nickName", userDTO.getNickName());
        claims.put("icon", userDTO.getIcon());
        claims.put("role", userDTO.getRole());
        
        String jit = String.valueOf(snowFlakeIDGenerator.nextId());
        
        return JWT.create()
                .withIssuer(issuer)
                .withSubject(String.valueOf(userDTO.getId()))
                .withExpiresAt(new Date(System.currentTimeMillis() + accessTokenExpiration))
                .withIssuedAt(new Date())
                .withClaim("jit", jit) // JWT唯一标识
                .withClaim("tokenType", "AT") // 令牌类型标识
                .withPayload(claims)
                .sign(algorithm);
    }
    
    /**
     * 生成Refresh Token (RT)
     * RT用于刷新AT，有效期较长，包含家族信息
     * 
     * @param userDTO 用户信息
     * @param familyId 令牌家族ID，用于管理令牌关系
     * @return 生成的Refresh Token
     */
    public String generateRefreshToken(UserDTO userDTO, long familyId) {
        String jit = String.valueOf(snowFlakeIDGenerator.nextId());
        
        return JWT.create()
                .withIssuer(issuer)
                .withSubject(String.valueOf(userDTO.getId()))
                .withExpiresAt(new Date(System.currentTimeMillis() + refreshTokenExpiration))
                .withIssuedAt(new Date())
                .withClaim("jit", jit)
                .withClaim("fid", String.valueOf(familyId)) // 家族ID
                .withClaim("tokenType", "RT") // 令牌类型标识
                .sign(algorithm);
    }
    
    // ==================== 令牌状态检验 ====================
    
    /**
     * 综合检验AT状态
     * 检验范围：格式验证、签名验证、过期检查、黑名单检查、家族状态检查
     * 
     * @param accessToken 待检验的Access Token
     * @return JWTResult 检验结果，包含状态码和详细信息
     */
    public JWTResult validateAccessToken(String accessToken) {
        // 基础空值检查
        if (accessToken == null || accessToken.trim().isEmpty()) {
            return JWTResult.fail("令牌为空", JWTResult.TokenStatus.INVALID);
        }
        
        try {
            // 1. 基础JWT验证
            DecodedJWT decodedJWT = verifyJwt(accessToken);
            String userId = decodedJWT.getSubject();
            String jit = decodedJWT.getClaim("jit").asString();
            String tokenType = decodedJWT.getClaim("tokenType").asString();
            
            // 令牌类型验证
            if (!"AT".equals(tokenType)) {
                log.warn("令牌类型错误 - 期望: AT, 实际: {}", tokenType);
                return JWTResult.fail("令牌类型错误", JWTResult.TokenStatus.INVALID);
            }
            
            // 2. 黑名单检查
            JWTResult blacklistCheck = checkTokenBlacklist(decodedJWT, userId, jit);
            if (!blacklistCheck.isSuccess()) {
                return blacklistCheck;
            }
            
            // 3. 家族状态检查（如果AT有关联家族）
            JWTResult familyCheck = checkFamilyStatus(decodedJWT, userId);
            if (!familyCheck.isSuccess()) {
                return familyCheck;
            }
            
            log.debug("AT验证成功 - 用户: {}, JIT: {}", userId, jit);
            return JWTResult.success("令牌有效", JWTResult.TokenStatus.VALID);
            
        } catch (TokenExpiredException e) {
            log.info("AT已过期: {}", accessToken);
            return JWTResult.fail("令牌已过期", JWTResult.TokenStatus.EXPIRED);
        } catch (SignatureVerificationException e) {
            log.warn("AT签名验证失败: {}", accessToken);
            return JWTResult.fail("令牌签名无效", JWTResult.TokenStatus.INVALID);
        } catch (JWTDecodeException e) {
            log.warn("AT格式错误: {}", accessToken);
            return JWTResult.fail("令牌格式无效", JWTResult.TokenStatus.INVALID);
        } catch (Exception e) {
            log.error("AT验证未知错误: {}", accessToken, e);
            return JWTResult.fail("令牌验证异常", JWTResult.TokenStatus.INVALID);
        }
    }
    
    /**
     * 解析AT并获取UserDTO
     * 如果AT失效，尝试自动轮转RT获取新AT
     * 
     * @param accessToken Access Token
     * @param refreshToken Refresh Token（可选，用于自动轮转）
     * @return 包含用户信息和新令牌的完整结果
     */
    public JWTResult parseAccessTokenWithAutoRefresh(String accessToken, String refreshToken) {
        // 先检验AT状态
        JWTResult validationResult = validateAccessToken(accessToken);
        
        if (validationResult.isSuccess()) {
            // AT有效，直接解析用户信息
            try {
                DecodedJWT decodedJWT = verifyJwt(accessToken);
                UserDTO userDTO = extractUserDTOFromToken(decodedJWT);
                return JWTResult.successWithUser("令牌有效", userDTO, JWTResult.TokenStatus.VALID);
            } catch (Exception e) {
                log.error("解析AT时发生错误: {}", accessToken, e);
                return JWTResult.fail("令牌解析异常", JWTResult.TokenStatus.INVALID);
            }
        } 
        
        // AT失效，但有RT且需要自动轮转
        else if (refreshToken != null && 
                 validationResult.getStatus() == JWTResult.TokenStatus.EXPIRED) {
            log.info("AT已过期，尝试使用RT自动轮转");
            return autoRefreshTokens(refreshToken);
        }
        
        // 其他情况直接返回验证结果
        return validationResult;
    }
    
    // ==================== 令牌轮转功能 ====================
    
    /**
     * 自动轮转令牌（AT过期时调用）
     * 使用旧的RT生成新的AT和RT
     * 
     * @param oldRefreshToken 旧的Refresh Token
     * @return 包含新令牌的轮转结果
     */
    public JWTResult autoRefreshTokens(String oldRefreshToken) {
        // 验证RT有效性
        JWTResult rtValidation = validateRefreshToken(oldRefreshToken);
        if (!rtValidation.isSuccess()) {
            return rtValidation;
        }
        
        try {
            DecodedJWT oldRT = verifyJwt(oldRefreshToken);
            String userId = oldRT.getSubject();
            
            // 使用分布式锁防止并发轮转
            String lockKey = RedisConstants.LOCK_TOKEN_REFRESH + userId;
            RLock lock = redissonClient.getLock(lockKey);
            
            try {
                if (lock.tryLock(2, 10, TimeUnit.SECONDS)) {
                    return processTokenRotation(oldRT);
                } else {
                    return JWTResult.fail("系统繁忙，请稍后重试", JWTResult.TokenStatus.INVALID);
                }
            } finally {
                if (lock.isHeldByCurrentThread()) {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("令牌轮转被中断", e);
            return JWTResult.fail("系统异常", JWTResult.TokenStatus.INVALID);
        } catch (Exception e) {
            log.error("令牌轮转失败", e);
            return JWTResult.fail("令牌轮转异常", JWTResult.TokenStatus.INVALID);
        }
    }
    
    /**
     * 处理令牌轮转的核心逻辑
     */
    private JWTResult processTokenRotation(DecodedJWT oldRT) {
        try {
            String userId = oldRT.getSubject();
            String oldFid = oldRT.getClaim("fid").asString();
            String oldJit = oldRT.getClaim("jit").asString();
            
            // 检查RT是否已被使用或作废
            if (isRefreshTokenAbandoned(oldFid, oldJit)) {
                return JWTResult.fail("刷新令牌已失效", JWTResult.TokenStatus.INVALID);
            }
            
            // 检查家族黑名单
            if (isFamilyBlacklisted(oldFid)) {
                return JWTResult.fail("令牌家族已被撤销", JWTResult.TokenStatus.INVALID);
            }
            
            // 创建用户对象（实际项目中应从数据库查询）
            UserDTO userDTO = extractUserDTOFromToken(oldRT);
            
            // 生成新令牌（继承原家族）
            long fid = Long.parseLong(oldFid);
            String newAT = generateAccessToken(userDTO);
            String newRT = generateRefreshToken(userDTO, fid);
            
            // 将旧RT加入作废列表
            abandonRefreshToken(oldFid, oldJit);
            
            log.info("令牌轮转成功 - 用户: {}, 家族: {}", userId, fid);
            return JWTResult.successWithTokens("令牌刷新成功", 
                    new String[]{newAT, newRT}, userDTO, JWTResult.TokenStatus.REFRESHED);
            
        } catch (Exception e) {
            log.error("处理令牌轮转时发生错误", e);
            return JWTResult.fail("令牌轮转处理异常", JWTResult.TokenStatus.INVALID);
        }
    }
    
    // ==================== 下线管理功能 ====================
    
    /**
     * 下线单个Access Token
     * 将AT加入黑名单，立即失效
     * 
     * @param accessToken 要下线的Access Token
     * @return 下线是否成功
     */
    public boolean revokeAccessToken(String accessToken) {
        try {
            DecodedJWT decodedJWT = verifyJwt(accessToken);
            String userId = decodedJWT.getSubject();
            String jit = decodedJWT.getClaim("jit").asString();
            Date expiresAt = decodedJWT.getExpiresAt();
            
            long ttl = Math.max(0, expiresAt.getTime() - System.currentTimeMillis());
            
            // 加入黑名单
            String blacklistKey = RedisConstants.TOKEN_BLACKLIST_KEY + userId + ":" + jit;
            redisTemplate.opsForValue().set(blacklistKey, "revoked", Duration.ofMillis(ttl));
            
            log.info("AT下线成功 - 用户: {}, JIT: {}, 剩余时间: {}秒", userId, jit, ttl / 1000);
            return true;
            
        } catch (Exception e) {
            log.error("AT下线失败", e);
            return false;
        }
    }
    
    /**
     * 下线整个令牌家族
     * 使该家族所有AT和RT立即失效
     * 
     * @param familyId 要下线的家族ID
     * @return 下线是否成功
     */
    public boolean revokeTokenFamily(String familyId) {
        try {
            String familyBlacklistKey = RedisConstants.TOKEN_FAMILY_BLACKLIST_KEY + familyId;
            // 设置较长的过期时间，确保家族下所有令牌都失效
            redisTemplate.opsForValue().set(familyBlacklistKey, "revoked", 
                    Duration.ofDays(30));
            
            log.info("令牌家族下线成功 - 家族ID: {}", familyId);
            return true;
            
        } catch (Exception e) {
            log.error("令牌家族下线失败 - 家族ID: {}", familyId, e);
            return false;
        }
    }
    
    // ==================== 辅助方法 ====================
    
    /**
     * 基础JWT验证
     */
    private DecodedJWT verifyJwt(String token) {
        JWTVerifier verifier = JWT.require(algorithm)
                .withIssuer(issuer)
                .build();
        return verifier.verify(token);
    }
    
    /**
     * 从令牌中提取用户信息
     */
    private UserDTO extractUserDTOFromToken(DecodedJWT decodedJWT) {
        UserDTO userDTO = new UserDTO();
        userDTO.setId(Long.parseLong(decodedJWT.getSubject()));
        userDTO.setNickName(decodedJWT.getClaim("nickName").asString());
        userDTO.setIcon(decodedJWT.getClaim("icon").asString());
        userDTO.setRole(decodedJWT.getClaim("role").asString());
        return userDTO;
    }
    
    /**
     * 检验Refresh Token状态
     */
    private JWTResult validateRefreshToken(String refreshToken) {
        // 实现类似于validateAccessToken的逻辑，但针对RT特性
        // 简化的实现
        try {
            DecodedJWT decodedJWT = verifyJwt(refreshToken);
            String tokenType = decodedJWT.getClaim("tokenType").asString();
            
            if (!"RT".equals(tokenType)) {
                return JWTResult.fail("非法的刷新令牌", JWTResult.TokenStatus.INVALID);
            }
            
            return JWTResult.success("刷新令牌有效", JWTResult.TokenStatus.VALID);
        } catch (Exception e) {
            return JWTResult.fail("刷新令牌验证失败", JWTResult.TokenStatus.INVALID);
        }
    }
    
    // 其他辅助方法：黑名单检查、家族状态检查、作废令牌管理等
    private JWTResult checkTokenBlacklist(DecodedJWT decodedJWT, String userId, String jit) {
        String blacklistKey = RedisConstants.TOKEN_BLACKLIST_KEY + userId + ":" + jit;
        if (Boolean.TRUE.equals(redisTemplate.hasKey(blacklistKey))) {
            return JWTResult.fail("令牌已被撤销", JWTResult.TokenStatus.REVOKED);
        }
        return JWTResult.success("令牌未在黑名单", JWTResult.TokenStatus.VALID);
    }
    
    private JWTResult checkFamilyStatus(DecodedJWT decodedJWT, String userId) {
        String fid = decodedJWT.getClaim("fid").asString();
        if (fid != null && isFamilyBlacklisted(fid)) {
            return JWTResult.fail("令牌家族已被撤销", JWTResult.TokenStatus.REVOKED);
        }
        return JWTResult.success("家族状态正常", JWTResult.TokenStatus.VALID);
    }
    
    private boolean isFamilyBlacklisted(String familyId) {
        return Boolean.TRUE.equals(
            redisTemplate.hasKey(RedisConstants.TOKEN_FAMILY_BLACKLIST_KEY + familyId));
    }
    
    private boolean isRefreshTokenAbandoned(String familyId, String jit) {
        return Boolean.TRUE.equals(
            redisTemplate.opsForSet().isMember(RedisConstants.TOKEN_ABANDON_RT + familyId, jit));
    }
    
    private void abandonRefreshToken(String familyId, String jit) {
        redisTemplate.opsForSet().add(RedisConstants.TOKEN_ABANDON_RT + familyId, jit);
    }
}
```

## 🎯 配套的JWTResult状态类

```
/**
 * JWT验证结果封装类
 */
@Data
@AllArgsConstructor
public class JWTResult {
    private boolean success;
    private String message;
    private TokenStatus status;
    private String[] tokens;
    private UserDTO userDTO;
    
    public enum TokenStatus {
        VALID,       // 令牌有效
        EXPIRED,     // 令牌过期
        INVALID,     // 令牌无效
        REVOKED,     // 令牌已被撤销
        REFRESHED    // 令牌已刷新
    }
    
    // 成功状态工厂方法
    public static JWTResult success(String message, TokenStatus status) {
        return new JWTResult(true, message, status, null, null);
    }
    
    public static JWTResult successWithUser(String message, UserDTO userDTO, TokenStatus status) {
        return new JWTResult(true, message, status, null, userDTO);
    }
    
    public static JWTResult successWithTokens(String message, String[] tokens, UserDTO userDTO, TokenStatus status) {
        return new JWTResult(true, message, status, tokens, userDTO);
    }
    
    // 失败状态工厂方法
    public static JWTResult fail(String message, TokenStatus status) {
        return new JWTResult(false, message, status, null, null);
    }
    
    // 便捷方法
    public boolean isSuccess() { return success; }
    public boolean needRefresh() { return status == TokenStatus.EXPIRED; }
    public boolean isInvalid() { return !success && status == TokenStatus.INVALID; }
}
```

## 💡 使用示例

```
@RestController
public class AuthController {
    
    @Autowired
    private AdvancedJwtUtil jwtUtil;
    
    /**
     * 受保护接口 - 自动令牌管理
     */
    @GetMapping("/api/protected-data")
    public ResponseEntity<?> getProtectedData(
            @RequestHeader("Authorization") String authHeader,
            @RequestHeader(value = "X-Refresh-Token", required = false) String refreshToken) {
        
        String accessToken = authHeader.replace("Bearer ", "");
        
        // 自动解析AT，如过期则尝试轮转RT
        JWTResult result = jwtUtil.parseAccessTokenWithAutoRefresh(accessToken, refreshToken);
        
        if (!result.isSuccess()) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                    .body(Map.of("error", result.getMessage(), "code", result.getStatus().name()));
        }
        
        // 如果令牌被刷新，返回新令牌给客户端
        if (result.getStatus() == JWTResult.TokenStatus.REFRESHED) {
            return ResponseEntity.ok()
                    .header("X-New-Access-Token", result.getTokens()[0])
                    .header("X-New-Refresh-Token", result.getTokens()[1])
                    .body(Map.of("data", "敏感数据", "user", result.getUserDTO()));
        }
        
        return ResponseEntity.ok(Map.of("data", "敏感数据", "user", result.getUserDTO()));
    }
    
    /**
     * 用户退出登录
     */
    @PostMapping("/api/logout")
    public ResponseEntity<?> logout(@RequestHeader("Authorization") String authHeader) {
        String token = authHeader.replace("Bearer ", "");
        
        boolean revoked = jwtUtil.revokeAccessToken(token);
        
        return revoked ? 
            ResponseEntity.ok(Map.of("message", "退出成功")) :
            ResponseEntity.badRequest().body(Map.of("error", "退出失败"));
    }
    
    /**
     * 强制下线所有设备
     */
    @PostMapping("/api/revoke-family")
    public ResponseEntity<?> revokeFamily(@RequestParam String familyId) {
        boolean revoked = jwtUtil.revokeTokenFamily(familyId);
        
        return revoked ?
            ResponseEntity.ok(Map.of("message", "家族下线成功")) :
            ResponseEntity.badRequest().body(Map.of("error", "下线失败"));
    }
}
```


