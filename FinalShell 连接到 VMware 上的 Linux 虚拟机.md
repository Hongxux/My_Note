好的，通过 FinalShell 连接到 VMware 上的 Linux 虚拟机是一个非常常见的操作，可以让你在 Windows/Mac 主机上用一个强大的图形化工具来管理虚拟机。

下面我为你整理了详细的步骤和示意图，确保你一定能成功连接。

### 连接成功的关键要素

要让 FinalShell 成功连接到虚拟机，必须满足以下**三个条件**，你可以对照流程图快速检查和定位问题：

---

### 步骤一：配置虚拟机网络（满足条件1：网络互通）

这是最关键的一步，决定了你的主机能否找到虚拟机。VMware 提供了几种网络模式，对于连接来说，**NAT模式**是最简单且通用的选择。

1. **确保虚拟机已关机**。
    
2. 在 VMware 中，选中你的 Linux 虚拟机，依次点击 **“虚拟机”**-> **“设置”**-> **“网络适配器”**。
    
3. 在右侧选择 **“NAT 模式”**，并勾选 **“启动时连接”**。然后点击“确定”。
    

#### 获取虚拟机的IP地址

1. 启动你的 Linux 虚拟机，并登录。
    
2. 打开终端，输入以下命令之一来查询虚拟机的 IP 地址：
    
    - **现代命令（推荐）**：
        
        ```
        ip address
        ```
        
    - 或传统的：
        
        ```
        ifconfig
        ```
        
    
3. 在输出信息中，找到 `ens33`、`eth0`等类似的网络接口名称，其 `inet`字段后面的地址就是虚拟机的 IP。通常格式为 `192.168.x.x`。
    
    **记下这个IP地址**，例如：`192.168.229.128`。
    

#### 【可选】设置静态IP（强烈推荐）

使用DHCP时，虚拟机的IP地址可能会变。设置为静态IP可以一劳永逸。

请根据你的虚拟机IP修改下面的配置，然后使用 `sudo netplan apply`命令应用。
**记住，多个配置只有一个生效**
```
# 编辑Netplan配置文件（文件名可能略有不同）
sudo vim /etc/netplan/00-installer-config.yaml

# 文件内容示例（使用你的实际IP和网关）
# 让 NetworkManager 管理系统上的所有设备  
network:  
  version: 2  
  # 指定使用 networkd 作为渲染器  
  renderer: networkd  
  ethernets:  
    # 您的网络接口名称，请确保与 'ip addr' 命令显示的一致  
    ens33:  
      # 禁用 DHCP，使用静态 IP  
      dhcp4: no  
      # 静态 IP 地址和子网掩码（CIDR 格式）  
      addresses: [192.168.0.10/24]  
      # 默认网关（您的路由器地址）  
      gateway4: 192.168.0.1  
      # DNS 服务器  
      nameservers:  
        addresses: [8.8.8.8, 114.114.114.114]  
      # 将此网卡设置为可选的，避免系统启动时因网络未就绪而卡住  
      optional: true  
# Let NetworkManager manage all devices on this system
```

---

### 步骤二：确保SSH服务已安装并启动（满足条件2）

大多数 Linux 发行版默认不安装 SSH 服务端。

1. 在虚拟机终端中，执行以下命令安装并启动 SSH 服务：
    
    ```
    # 1. 安装 openssh-server
    sudo apt update
    sudo apt install openssh-server
    
    # 2. 启动 SSH 服务
    sudo systemctl start ssh
    
    # 3. 设置开机自启
    sudo systemctl enable ssh
    
    # 4. 检查服务状态，看到 active (running) 表示成功
    sudo systemctl status ssh
    ```
    
出现拒绝连接 ，使用`sudo systemctl status ssh`检查出错原因
查看主机能够ping通虚拟机地址，说明主机和虚拟机的网络是通的，原因在于Ubantu一般会默认安装openssh-client，但是未安装openssh-server。  
首先在虚拟机终端输入命令sudo apt install openssh-server安装ssh服务器，输入命令sudo apt install openssh-client安装ssh客户端。然后输入命令 sudo gedit /etc/ssh/ssh_config配置ssh客户端，去掉PasswordAuthentication yes前面的#号，保存后退出，再输入命令sudo gedit /etc/ssh/sshd_config配置ssh服务器，在PermitRootLogin prohibit-password加一句PermitRootLogin yes（加了这个后在finalshell等可以直接root连接虚拟机，用root连接才可以上传文件），保存退出。此外端口号22前面可能有#，有的去掉。  
最后执行sudo /etc/init.d/ssh restart 重启ssh服务。

还有可能是因为root账户被锁定
- sudo passwd -S root检查root账户状态，如果被锁定
- 原因：Ubuntu 在安装后，**root 用户的密码处于一种未设置的特殊状态**，因此任何尝试用密码登录 root 的行为都会失败，这在现象上表现为"锁定" 。这样做的好处是引导用户使用 `sudo`来执行管理任务。`
- 解决方法
	1. **为root账户解锁并设置新密码**
    由于root账户被锁定，您需要先使用`sudo`权限为其解锁并重新设置一个密码。在终端中执行
    ```
    sudo passwd -u root
    ```
    这个命令会解锁root账户。解锁后，建议您立即为root账户设置一个新密码以确保安全
    
    ```
    sudo passwd root
    ```
    系统会提示您输入并确认新的root密码。
    
	2. **验证密码状态**
	    
	    设置新密码后，再次检查root账户的状态，确认已变为 `P`（Password set，密码已设置）
	    
	    ```
	    sudo passwd -S root
	    ```
	    
	    理想的输出应类似于：`root P 09/21/2025 0 99999 7 -1`

---

### 步骤三：配置防火墙（满足条件3）

如果虚拟机开启了防火墙，需要放行 SSH 的 22 端口。

```
# 开放 22 端口
sudo ufw allow ssh

# 或者直接指定端口
sudo ufw allow 22/tcp

# 启用防火墙（如果尚未启用）
sudo ufw enable

# 查看防火墙状态
sudo ufw status
```
```
sudo firewall-cmd --zone=public --add-port=22/tcp --permanent
sudo firewall-cmd --reload
```
---

### 步骤四：在 FinalShell 中创建连接

现在，所有准备工作已完成，可以在 FinalShell 中建立连接了。

1. 打开 FinalShell。
    
2. 点击左上角的 **“文件夹”**图标，打开连接管理器。
    
3. 点击 **“+”**号新建一个连接。
    
4. 在弹出的窗口中，按下图所示填写信息：
    
    - **名称**：给你的连接取个名字（如：My-Ubuntu-VM）
        
    - **主机**：填写你之前在虚拟机中查到的 **IP 地址**（如：192.168.229.128）
        
    - **端口**：`22`（默认SSH端口）
        
    - **用户名**：你的 Linux 虚拟机登录用户名（如：`ubuntu`）
        
    - **密码**：该用户对应的密码
        
        （如果你配置了密钥认证，可以选择“公钥”方式）
        
    
5. 点击 **“确定”**，然后双击这个新创建的连接。
    
6. 如果一切正常，你会看到命令行提示符，表示已经成功连接到虚拟机！
    

