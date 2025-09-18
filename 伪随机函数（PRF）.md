
#### 消除弱随机源风险**​

- ​**场景假设**​：
    
    若直接拼接随机源（`PremasterSecret+ClientRandom+ServerRandom`）作为密钥：
    
    - 若服务器随机数生成器存在缺陷（如熵不足），密钥熵值大幅降低。

### 一、**Master Secret的生成原理**​

#### 1. ​**输入参数完全一致**​

- ​**三方随机源**​：
    
    客户端与服务器各自持有以下**相同输入**​：
    
- ​**输入参数**​：
    
    - Premaster Secret`（客户端生成，经服务器公钥加密传输 → 服务器私钥解密获取）
        
    - `ClientRandom`（客户端生成，明文传输）
        
    - `ServerRandom`（服务器生成，明文传输）
    
- ​**PRF工作流程**​：
		MasterSecret=PRF(PremasterSecret,"mastersecret",ClientRandom+ServerRandom)
    
    伪随机函数将三个独立随机源**充分混合**，生成48字节`MasterSecret`。
    
- ​**安全价值**​：
    
    即使某随机源部分泄露（如`ClientRandom`被预测），PRF的不可逆性仍保障`MasterSecret`安全。
    

#### 2. ​**伪随机函数（PRF）的确定性**​

MasterSecret=PRF(PremasterSecret,"mastersecret",ClientRandom+ServerRandom)

- ​**PRF特性**​：相同的输入参数 ⇒ ​**必然输出相同结果**​（如HMAC-SHA256的确定性计算）。
    
- ​**协议强制**​：双方必须使用**完全相同的PRF算法**​（由握手协商的加密套件确定）。
    

---

### 二、**密钥一致性的协议保障**​

#### 1. ​**握手消息完整性校验**​

- ​**Finished报文验证**​：
    
    双方用生成的`MasterSecret`派生临时密钥 → 加密所有握手消息的哈希值 → 交换`Finished`报文。
    
    - ​**验证逻辑**​：若解密后的哈希值匹配，则证明：
        
        a) `MasterSecret`相同
        
        b) 握手过程未被篡改
        
    

#### 2. ​**失败场景与处理**​

|​**不一致原因**​|系统行为|触发条件示例|
|---|---|---|
|`PremasterSecret`解密失败|握手终止（`decrypt_error`警报）|证书私钥不匹配或传输损坏|
|`ClientRandom/ServerRandom`被篡改|`Finished`报文校验失败 → 连接中断|中间人攻击修改随机数|

---

