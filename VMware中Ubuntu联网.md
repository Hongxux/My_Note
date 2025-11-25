在VMware的Ubuntu虚拟机中配置桥接联网，能让您的虚拟机像一台独立的物理设备一样接入您的主机所在的局域网，非常适合需要对外提供服务或与局域网内其他设备互通的场景。🍀

下面为您梳理详细的配置步骤和关键要点。

### 🔧 配置步骤概要

配置过程主要分为两大步：先在VMware软件中进行设置，然后在Ubuntu虚拟机内配置网络。

| 步骤  | 操作位置               | 核心动作                | 说明                       |
| :-- | :----------------- | :------------------ | :----------------------- |
| 1   | VMware Workstation | 将虚拟机网络适配器设置为桥接模式    | 告诉VMware将此虚拟机直接连接到物理网络   |
| 2   | Ubuntu 虚拟机内        | 配置网络参数（IP地址、网关、DNS） | 使虚拟机获得网络身份，通常使用Netplan工具 |

### 📘 详细操作指南

#### 1. 在VMware中设置桥接模式

1.  确保虚拟机已关闭。
2.  在VMware Workstation中，选中您的Ubuntu虚拟机，然后依次点击 “虚拟机” -> “设置”。
3.  在弹出的窗口里，找到 “网络适配器”。
4.  在右侧的“网络连接”选项中，选择 “桥接模式”，并且务必勾选下方的 “复制物理网络连接状态”（这对于笔记本电脑在Wi-Fi和有线网络间切换时保持连接很有帮助）。
5.  （可选但推荐）点击VMware主菜单的 “编辑” -> “虚拟网络编辑器”。在弹出的窗口中，找到与桥接模式对应的 `VMnet0`，确保其“桥接到”的物理网卡是正确的（例如，如果您的主机用Wi-Fi上网，就选择无线网卡；用网线上网，就选择有线网卡）。

#### 2. 在Ubuntu虚拟机内配置网络（适用于18.04及之后版本）

现代Ubuntu版本使用 Netplan 来管理网络配置，配置文件通常是YAML格式。请特别注意YAML的语法格式（如缩进和冒号后的空格），这是导致配置失败的常见原因。

1.  打开终端，使用文本编辑器（如`nano`或`vim`）编辑Netplan配置文件。常见配置文件位于 `/etc/netplan/` 目录下，名称可能类似 `01-netcfg.yaml` 或 `00-installer-config.yaml`。您可以使用 `sudo ls /etc/netplan/` 命令查看具体文件名。
    ```bash
    sudo nano /etc/netplan/01-netcfg.yaml
    ```

2.  编辑配置文件。以下是一个基本的静态IP配置示例。请务必将 `addresses` 和 `gateway4` 等参数修改为您自己局域网的环境参数。
    ```yaml
    network:
      version: 2
      ethernets:
        ens33: # 这里填写你的网卡名称，可以使用 `ip a` 命令查看
          addresses:
            - 192.168.0.100/24 # 【重要】请将此IP改为您局域网内一个未被占用的IP地址
          gateway4: 192.168.0.1 # 【重要】修改为您局域网的路由器网关地址
          nameservers:
            addresses:
              - 8.8.8.8
              - 114.114.114.114 # 设置DNS服务器地址
    ```
    ⚠️ 关键提示：
    ◦   `addresses` 字段：YAML语法要求它必须是一个列表，即使只有一个IP地址，也需要用方括号 `[]` 括起来，并以短横线 `-` 开头。您之前遇到的错误就是因为直接写了字符串而非列表。

    ◦   IP地址选择：您为虚拟机设置的静态IP必须与您的主机IP在同一网段，且未被其他设备使用。您可以在主机上打开命令提示符（Windows）或终端（Mac/Linux），输入 `ipconfig`（Windows）或 `ifconfig`（Linux/Mac）来查看主机的IP地址和网关信息。


3.  保存文件并应用配置。
    ```bash
    sudo netplan apply
    ```

### 💡 重要提示与故障排查

•   使用DHCP（动态获取IP）：如果您希望虚拟机自动从路由器获取IP，配置会更简单。只需将 `dhcp4` 设置为 `yes`，并删除 `addresses` 和 `gateway4` 等手动设置即可。

    ```yaml
    network:
      version: 2
      renderer: networkd
      ethernets:
        ens33:
          dhcp4: yes
    ```
•   检查桥接服务：如果在VMware的网络适配器设置中找不到桥接选项，或提示错误，可能需要检查主机系统的网络连接属性，确保已为物理网卡安装并勾选了 “VMware Bridge Protocol” 服务。

•   验证连通性：配置完成后，在Ubuntu虚拟机中尝试 `ping` 您的路由器网关地址（如 `ping ### (@replace=0)


- 如果配置没有反应：
	- 使用`ip addr show ens33`检查网卡有没有上线
		- 如果没有上线（处于 `DOWN`（关闭）状态）
			```
					# 1. 启用ens33网卡,用于将网卡状态从 `DOWN`变为 `UP`（激活）
					sudo ip link set ens33 up
					
					# 2. 尝试通过DHCP自动获取IP地址,请求动态获取IP地址。`-v`参数可以显示详细过程
					sudo dhclient -v ens33
			```
		- 确保render是networkd而不是老版本的NetWorkManager