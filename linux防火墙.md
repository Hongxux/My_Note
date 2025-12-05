放行特地端口
```
sudo firewall-cmd --zone=public --add-port=22/tcp --permanent
sudo firewall-cmd --reload
```




| 操作类别        | 常用命令示例                                                                                                                        | 关键说明                 |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------- | -------------------- |
| **基本状态管理**​ | `systemctl start firewalld`  <br>`systemctl enable firewalld`                                                                 | 启动、设置开机自启服务          |
| **端口管理**​   | `firewall-cmd --permanent --add-port=80/tcp`  <br>`firewall-cmd --reload`                                                     | 永久开放端口后需重载配置生效       |
| **服务管理**​   | `firewall-cmd --permanent --add-service=http`                                                                                 | 通过服务名管理（如 http, ssh） |
| **高级规则**​   | `firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=192.168.1.100 port port=8080 protocol=tcp accept'` | 实现精细控制（如限制IP）        |
| **区域管理**​   | `firewall-cmd --get-default-zone`  <br>`firewall-cmd --set-default-zone=public`                                               | 查看及设置默认区域            |

### 🔧 详细操作说明

#### 1. **管理防火墙服务**

确保防火墙在系统启动时自动运行，是安全防护的第一步 。

```
# 启动防火墙
sudo systemctl start firewalld
# 设置防火墙开机自启
sudo systemctl enable firewalld
# 检查防火墙状态
sudo systemctl status firewalld
# 停止防火墙（谨慎使用）
sudo systemctl stop firewalld
```

#### 2. **管理端口与协议**

这是最常用的操作，允许或拒绝特定端口的网络流量。**切记，使用 `--permanent`参数后，必须执行 `--reload`才能使永久规则生效**​ 。

```
# 永久开放特定端口（如TCP端口8080）
sudo firewall-cmd --permanent --zone=public --add-port=8080/tcp
# 永久移除已开放的端口
sudo firewall-cmd --permanent --zone=public --remove-port=8080/tcp
# 批量开放连续端口范围（如UDP端口5000至5010）
sudo firewall-cmd --permanent --zone=public --add-port=5000-5010/udp
# 重新加载配置，使永久规则生效
sudo firewall-cmd --reload
# 查询端口是否开放
sudo firewall-cmd --zone=public --query-port=8080/tcp
```

#### 3. **通过服务名管理规则**

`firewalld`内置了许多常见服务（如 `http`, `https`, `ssh`, `ftp`）的定义，使用服务名比直接记端口号更方便，也更不易出错 。

```
# 永久开放HTTP服务（相当于开放80端口）
sudo firewall-cmd --permanent --zone=public --add-service=http
# 永久移除HTTP服务
sudo firewall-cmd --permanent --zone=public --remove-service=http
# 查看所有预定义服务
sudo firewall-cmd --get-services
# 查看当前区域已开放的服务
sudo firewall-cmd --zone=public --list-services
```

#### 4. **使用高级规则**

对于更复杂的需求，如限制特定源IP地址访问，可以使用功能强大的“富规则”（Rich Rules） 。

```
# 只允许IP 192.168.1.100访问本机的TCP 8080端口
sudo firewall-cmd --permanent --zone=public --add-rich-rule='rule family="ipv4" source address="192.168.1.100" port port="8080" protocol="tcp" accept'
# 移除该条富规则
sudo firewall-cmd --permanent --zone=public --remove-rich-rule='rule family="ipv4" source address="192.168.1.100" port port="8080" protocol="tcp" accept'
```

#### 5. **理解与管理区域**

区域是 `firewalld`的一个核心概念，它为不同的网络环境（如不信任的公共场所、可信任的家庭网络）预设了不同的安全级别。将网络接口分配到合适的区域可以简化管理 。

```
# 查看默认区域和所有活跃区域
sudo firewall-cmd --get-default-zone
sudo firewall-cmd --get-active-zones
# 将网络接口enp0s3的默认区域设置为dmz
sudo firewall-cmd --permanent --change-zone=enp0s3 --zone=dmz
```

### 💡 最佳实践与注意事项

- **最小权限原则**：只开放业务所必需的最少端口，关闭所有不必要的端口和服务 。
    
- **规则持久化**：进行生产环境变更时，务必加上 `--permanent`参数，并在修改后执行 `--reload`，这样配置才能在重启后依旧有效 。
    
- **操作前检查**：在添加或移除规则前，使用 `--list-all`命令查看当前所有规则，做到心中有数 。
    
- **组合使用**：可以混合使用服务、端口和富规则，以满足复杂的访问控制需求。
    

希望这份总结能帮助你更自信地管理服务器防火墙！如果你对某个具体操作场景有更多疑问，我很乐意提供进一步的解答。