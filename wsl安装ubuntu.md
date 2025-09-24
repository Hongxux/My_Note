## wsl介绍
**功能**：开发人员可以在 Windows 计算机上同时访问 Windows 和 Linux 的强大功能。 借助适用于 Linux 的 Windows 子系统（WSL），开发人员可以安装 Linux 分发版（如 Ubuntu、OpenSUSE、Kali、Debian、Arch Linux 等），并在 Windows 上直接使用 Linux 应用程序、实用工具和 Bash 命令行工具（未经修改）
**虚拟机的替代**：无需传统虚拟机或双包设置的开销。

## 安装教程
- 在搜索栏搜 powershell，管理员身份打开。
- 输入下面四条命令

```powershell
wsl --install
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --set-default-version 2
```
- 重启电脑
- ![[Pasted image 20250920170903.png]]
##### 出现问题
Error code: Wsl/Service/CreateInstance/MountVhd/HCS/ERROR_FILE_NOT_FOUND

## 安装ubuntu
- 下载appx
**Ubuntu2204：**  
[https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-220117.appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-220117.appx)  
[https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-220117_ARM64.appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-220117_ARM64.appx)  
**Ubuntu1804：**  
[https://wsldownload.azureedge.net/Ubuntu1804-230608_x64.appx](https://link.zhihu.com/?target=https%3A//wsldownload.azureedge.net/Ubuntu1804-230608_x64.appx) [https://wsldownload.azureedge.net/Ubuntu1804-230608_ARM64.appx](https://link.zhihu.com/?target=https%3A//wsldownload.azureedge.net/Ubuntu1804-230608_ARM64.appx)  
**Ubuntu2004：**  
[https://wsldownload.azureedge.net/Ubuntu2004-230608_x64.appx](https://link.zhihu.com/?target=https%3A//wsldownload.azureedge.net/Ubuntu2004-230608_x64.appx) [https://wsldownload.azureedge.net/Ubuntu2004-230608_ARM64.appx](https://link.zhihu.com/?target=https%3A//wsldownload.azureedge.net/Ubuntu2004-230608_ARM64.appx)  
**Ubuntu2204 LTS：**  
[https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204LTS-230518_x64.appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204LTS-230518_x64.appx) [https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204LTS-230518_ARM64.appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204LTS-230518_ARM64.appx)  
  
**Debian GNULinux_1.12.2.0：**[https://wsldownload.azureedge.net/TheDebianProject.DebianGNULinux_1.12.2.0_neutral___76v4gfsz19hv4.AppxBundle](https://link.zhihu.com/?target=https%3A//wsldownload.azureedge.net/TheDebianProject.DebianGNULinux_1.12.2.0_neutral___76v4gfsz19hv4.AppxBundle)  
[https://wsldownload.azureedge.net/TheDebianProject.DebianGNULinux_1.12.2.0_neutral___76v4gfsz19hv4.AppxBundle](https://link.zhihu.com/?target=https%3A//wsldownload.azureedge.net/TheDebianProject.DebianGNULinux_1.12.2.0_neutral___76v4gfsz19hv4.AppxBundle)  
  
**KaliLinux_1.13.1.0：**  
[https://wsldownload.azureedge.net/KaliLinux_1.13.1.0.AppxBundle](https://link.zhihu.com/?target=https%3A//wsldownload.azureedge.net/KaliLinux_1.13.1.0.AppxBundle)  
  
**OracleLinux：**  
[https://wslstorestorage.blob.core.windows.net/wslblob/OracleLinux_7.9-230428.Appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/OracleLinux_7.9-230428.Appx)  
[https://wslstorestorage.blob.core.windows.net/wslblob/OracleLinux_8.7-230428.Appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/OracleLinux_8.7-230428.Appx)  
**openSuseLeap15-5：**  
[https://wslstorestorage.blob.core.windows.net/wslblob/OracleLinux_9.1-230428.Appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/OracleLinux_9.1-230428.Appx)  
[https://wslstorestorage.blob.core.windows.net/wslblob/openSuseLeap15-5-231506_x64.Appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/openSuseLeap15-5-231506_x64.Appx)  
**SUSELinuxEnterpriseServer15：**  
[https://wslstorestorage.blob.core.windows.net/wslblob/SUSELinuxEnterpriseServer15_SP4-230130_x64.Appx](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/SUSELinuxEnterpriseServer15_SP4-230130_x64.Appx)  
[https://wslstorestorage.blob.core.windows.net/wslblob/SUSELinuxEnterpriseServer15_SP5-](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/SUSELinuxEnterpriseServer15_SP5-)  
230628.Appx  
**openSUSETumbleweed：**  
[https://wslstorestorage.blob.core.windows.net](https://link.zhihu.com/?target=https%3A//wslstorestorage.blob.core.windows.net/wslblob/openSUSETumbleweed-230130_x64.Appx)

- 在设置中打开开发者选项
![[Pasted image 20250911203910.png]]

- 运行下载的软件
![[Pasted image 20250911203927.png]]
![[Pasted image 20250911203942.png]]

> [!NOTE] Title
>**一个是username的规则：**
> - 以小写字母开头。
>- 只能包含小写字母、数字和下划线 `_`。
>- 不能包含大写字母、空格、特殊字符（如 `@`, `$`, `#`, `%`, `&`, `-`等）
>- 长度可能也有一定限制。
>
> **一个是密码输入没有显示，这个是保护，不然别人看到你输入的密码，实际上是输入进去了**
> 密码我的是h3800168

## visual studio code连接wsl
在扩展中安装wsl
![[Pasted image 20250911210108.png]]
## 配置csapp的lab环境
```wsl
wget https://gitee.com/lin-xi-269/csapplab/raw/origin/installAll.sh # 下载脚本
bash installAll.sh # 运行脚本
```
```bash
# 下载镜像
docker pull linxi177229/csapp:latest
# 查看
docker images
# 启动容器（里面有配置好的环境 和 PDF 资料）
docker run --name csapp -itd linxi177229/csapp 
# 进入容器 
docker attach csapp

# 接下来就和使用 平常的 Ubuntu：20.04 一样了
# 进入 lab1 进行一个简单的测试
cd ~
ls
cd csapplab
cd datalab/datalab-handout
make clean && make && ./btest
```