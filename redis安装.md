在Ubuntu虚拟机中安装Redis主要有通过**包管理器安装**和通过**源码编译安装**两种方式。包管理器安装最快捷，适合大多数需要快速部署的场景；源码安装则更灵活，适合需要特定版本或自定义编译选项的情况。

|特性|包管理器安装 (apt)|源码编译安装|
|---|---|---|
|**易用性**|**非常高**，几条命令即可完成|中等，需手动执行更多步骤|
|**版本**|软件源中的版本，可能非最新|**可安装任意版本**，包括最新版|
|**控制度**|标准配置，定制选项有限|**高**，可自定义编译参数和配置|
|**依赖管理**|自动处理|需手动安装部分依赖（如build-essential, tcl）|
|**推荐场景**|**新手推荐**，追求快速部署和简单性|需要特定版本、高级定制或学习编译过程|

### 🔧 安装方法与步骤

#### 方式一：通过apt包管理器安装（推荐新手）

这是最直接的方法，适合希望快速完成安装的用户。

1. **更新软件包列表**：打开终端，执行以下命令以确保获取最新的软件源信息。
    
    ```
    sudo apt update
    ```
    
2. **安装Redis服务器**：
    
    ```
    sudo apt install redis-server
    ```
    
    此命令会从Ubuntu官方软件源下载并安装Redis及其依赖。
    
3. **验证安装**：安装完成后，Redis服务通常会自动启动。可以通过以下命令检查其状态：
    
    ```
    sudo systemctl status redis-server
    ```
    
    如果服务正在运行，你会看到类似 `active (running)`的提示。
    
4. **基本功能测试**：使用Redis命令行客户端发送一个`PING`命令，如果Redis正常运行，它会返回`PONG`。
    
    ```
    redis-cli ping
    ```
    

#### 方式二：通过源码编译安装（适合有特定需求）

### 源码编译安装（获取最新版）

如果仓库中的版本不符合要求，或者您希望完全自定义安装，可以手动编译安装

。这种方法能获得最新版本，但步骤稍多。

1. **安装编译工具和依赖**：
    

    
    ```
    sudo dnf groupinstall "Development Tools" -y
    sudo dnf install wget -y
    ```
    
2. **下载并解压 Redis 源码**（请访问 Redis.io获取最新稳定版链接）：
    

    ```
    wget https://download.redis.io/releases/redis-7.2.4.tar.gz  # 版本号可替换
    tar xzf redis-7.2.4.tar.gz
    cd redis-7.2.4
    ```
    
3. **编译安装**：
    
    ```
    make
    sudo make install
    ```
### ⚙️ 安装后的基本配置

安装完成后，可能需要进行一些基本配置，比如设置密码和允许远程连接。

1. **设置认证密码**（可选但建议）：
    
    编辑Redis的配置文件，通常位于 `/etc/redis/redis.conf`。
    
    ```
    sudo nano /etc/redis/redis.conf
    ```
    
    在配置文件中找到 `# requirepass foobared`这一行，取消注释（删除行首的`#`），并将`foobared`替换为您自己的强密码。
    
    ```
    requirepass your_strong_password_here
    ```
    
2. **配置远程访问**（如果需要）：
    
    默认情况下，Redis只允许本地连接。要允许其他机器访问，需在同一配置文件中找到 `bind 127.0.0.1 ::1`这一行，将其注释掉（在行首加`#`），或者修改为 `bind 0.0.0.0`（意味着绑定到所有网络接口）。**请注意，这样配置后务必设置强密码，否则有安全风险**。
    
    ```
    # bind 127.0.0.1 ::1
    或
    bind 0.0.0.0
    ```
    
    同时，建议将保护模式（`protected-mode`）设置为 `no`。
    
    ```
    protected-mode no
    ```
    
3. **重启Redis以使配置生效**：对配置文件的任何修改后，都需要重启Redis服务。
    
    ```
    sudo systemctl restart redis-server  # 如果使用apt安装
    # 如果通过源码安装并以非服务方式运行，需要先结束进程再重新启动
    # redis-server /path/to/your/redis.conf &
    ```
```
    redis-server /etc/redis/redis.conf
```
    `sudo netstat -tulnp \| grep 6379`查看端口占用
`sudo pkill -9 redis-server`并 `sudo rm -f /var/run/redis_6379.pid`
### 🛠️ 常用管理与操作命令

- **启动Redis服务**：`sudo systemctl start redis-server`
    
- **停止Redis服务**：`sudo systemctl stop redis-server`
    
- **重启Redis服务**：`sudo systemctl restart redis-server`
    
- **设置开机自启**：`sudo systemctl enable redis-server`
    
- **连接Redis客户端**：`redis-cli`（如果设置了密码，连接后需执行 `AUTH yourpassword`）
    
- **测试连接**：在`redis-cli`中执行 `ping`，成功会返回 `PONG`。
    

### 💡 补充说明

- **防火墙**：如果虚拟机启用了防火墙（如`ufw`）并需要允许外部连接，记得放行Redis的默认端口（6379）：
    
    ```
    sudo ufw allow 6379
    ```
```
# 检查防火墙状态
sudo firewall-cmd --state

# 如果防火墙是运行状态，添加规则开放Redis端口
sudo firewall-cmd --permanent --add-port=6381/tcp
sudo firewall-cmd --reload

# 验证端口是否已开放
sudo firewall-cmd --list-ports | grep 6379
```
- **连接测试**：配置允许远程连接后，可以在宿主机或其他机器上使用`redis-cli -h <虚拟机IP> -p 6379`尝试连接。
    

希望这份详细的指南能帮助您顺利在Ubuntu虚拟机中完成Redis的安装和配置。