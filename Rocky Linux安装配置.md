### 安装系统
在创建虚拟机的时候，版本要选择Linux6.x不然会灰屏幕，安装不进去
![[Pasted image 20251122210845.png]]
### 配置静态ip
#### 使用 `nmcli`命令（推荐）

这种方法通过命令行直接修改配置，是官方推荐的方式。

1. **查看当前连接名称**
    
    首先，你需要知道要配置的网络连接叫什么。打开终端，输入以下命令：

    ```
    nmcli connection show
    ```
    
    输出结果中 `NAME`列就是连接名，通常是 `ens33`或 `ens160`等。记下它，后面会用到
    
    
2. **修改IP配置**
    
    将下面命令中的 `ens33`替换为你实际的连接名，并将IP地址、网关等参数替换成你网络环境中的值

    
    ```
    sudo nmcli connection modify "ens33" \
    ipv4.method manual \
    ipv4.addresses "192.168.0.100/24" \
    ipv4.gateway "192.168.0.1" \
    ipv4.dns "8.8.8.8,114.114.114.114" \
    ipv4.ignore-auto-dns yes
    ```
    
    - `ipv4.method manual`：表示设置为静态IP
        
    - `ipv4.addresses "192.168.0.100/24"`：设置IP和子网掩码。`/24`对应 `255.255.255.0`

    - `ipv4.gateway "192.168.0.1"`：设置网关。
        
    - `ipv4.dns "8.8.8.8,114.114.114.114"`：设置DNS服务器。
        
    - `ipv4.ignore-auto-dns yes`：确保只使用你设置的DNS
        

1. **应用配置**
    
    执行以下命令使新的网络配置生效
    
    ```
    sudo nmcli connection down "ens33" && sudo nmcli connection up "ens33"
    ```
### 配置防火墙
| 操作类型      | 命令                                    | 效果                            |
| --------- | ------------------------------------- | ----------------------------- |
| **临时关闭**​ | `sudo systemctl stop firewalld`       | 立即停止防火墙，但**系统重启后会自动恢复**。      |
| **永久禁用**​ | `sudo systemctl disable firewalld`    | 禁止防火墙开机自启，需与 `stop`命令结合使用。    |
| **状态检查**​ | `sudo systemctl status firewalld`     | 查看防火墙当前状态（是否活跃）。              |
| **确认禁用**​ | `sudo systemctl is-enabled firewalld` | 检查是否已成功禁用开机启动（应显示 `disabled`） |
