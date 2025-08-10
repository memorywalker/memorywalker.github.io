---
title: Widnows10中WSL使用Ubuntu
date: 2025-08-07 23:10:25
categories:
- linux
tags:
- wsl
- ubuntu
- windows
---

##  Windows10 使用WSL2运行Ubuntu

### 系统配置

#### 安装流程

1. 安装WSL，打开系统设置-应用与功能-Windows 功能，勾选其中的`Virtual Machine Platform`和`Windows Subsystem for Linux`，重启电脑

2. 到[install-manual](https://learn.microsoft.com/en-us/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package) 下载[WSL2 Linux kernel update package for x64 machines](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)，并安装

3. PowerShell中执行`wsl --set-default-version 2`设置使用WSL2

4. ubuntu[官网](https://releases.ubuntu.com/noble/) 下载24.04 LTS的WSL的镜像文件[64-bit PC (AMD64) WSL image](https://releases.ubuntu.com/noble/ubuntu-24.04.3-wsl-amd64.wsl)，得到文件`ubuntu-24.04.3-wsl-amd64.wsl`

5. 把这个文件解压后得到1.3GB的`ubuntu-24.04.3-wsl-amd64`文件

6. 使用wsl导入系统镜像到指定目录`wsl --import <系统名称> <安装位置> <镜像文件路径>`
```powershell
wsl --import Ubuntu-24.04 "E:\wsl\Ubuntu-24.04" "E:\wsl\ubuntu-24.04.3-wsl-amd64"

wsl.exe --import <Distro> <InstallLocation> <FileName> [Options]
Options:
    --version <Version>
    --vhd
```

   安装完成后会在`E:\wsl\Ubuntu-24.04`目录中生成一个`ext4.vhdx`文件，大小为1.5G多

7. 使用` wsl --list --all`查看当前已经安装的系统

```bash
PS C:\Users\Edison> wsl --list --all
Windows Subsystem for Linux Distributions:
Ubuntu-24.04 (Default)
```

8. 运行系统`wsl`因为只有一个子系统可以不用带其他参数，也可以指定系统`wsl -d Ubuntu-24.04`

```bash
 PS C:\Users\Edison> wsl
Windows Subsystem for Linux is now available in the Microsoft Store!
You can upgrade by running 'wsl.exe --update' or by visiting https://aka.ms/wslstorepage
Installing WSL from the Microsoft Store will give you the latest WSL updates, faster.
For more information please visit https://aka.ms/wslstoreinfo

Welcome to Ubuntu 24.04.3 LTS (GNU/Linux 5.10.16.3-microsoft-standard-WSL2 x86_64)

* Documentation:  https://help.ubuntu.com
* Management:     https://landscape.canonical.com
* Support:        https://ubuntu.com/pro

System information as of Thu Aug  7 23:27:19 CST 2025

System load:  0.08               Processes:             9
Usage of /:   0.5% of 250.98GB   Users logged in:       0
Memory usage: 1%                 IPv4 address for eth0: 172.25.129.208
Swap usage:   0%

This message is shown once a day. To disable it please create the
/root/.hushlogin file.   
```

#### 常用命令

* 查看当前系统状态， 在powershell中执行`wsl -l -v`
* 使用root用户登录，在powershell中执行`wsl -u -root`或者`wsl --distribution <Distribution Name> --user <User Name>`
* 帮助信息`wsl --help`
* 关闭系统`wsl --shutdown` 或者`wsl -t <系统名称>`

#### 文件访问

##### windows访问ubuntu系统文件

在windows资源管理器的地址栏输入`\\wsl$`，可以看到一个发行版名称的挂在目录

##### ubuntu访问windows目录

直接在终端下访问`/mnt/<windows盘符>`，例如`cd /mnt/e`就可以切换到windows的e盘下

### 系统使用

#### 修改系统源

把ubuntu.sources备份一个后，使用Vim修改里面的内容

```bash
cd /etc/apt/sources.list.d/
cp ubuntu.sources ubuntu.sources_bak
vim ubuntu.sources
```

文件中一共有两段内容，把其中官网地址都改为Aliyun的地址`http://mirrors.aliyun.com/ubuntu/`，其他不用变

```yaml
Types: deb
URIs: http://mirrors.aliyun.com/ubuntu/
Suites: noble noble-updates noble-security
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

更新软件信息`sudo apt-get update`

#### 新增一个用户

* 新增用户`adduser walker`，过程中按提示设置密码

* 新增用户默认是user用户组，如果以后要执行管理员权限命令，需要增加到sudo组中 `usermod -aG sudo walker`

* 查看用户的用户组`groups walker`

* 修改wsl的默认登录用户为waker，root账户下在`/etc/wsl.conf`文件中添加以下内容

  ```yaml
  [user]
  default=walker
  ```

### AMD 显卡驱动

#### 安装显卡驱动

amd官方指南文档 https://rocm.docs.amd.com/projects/radeon/en/latest/docs/install/wsl/install-radeon.html

1. 下载地址https://www.amd.com/zh-cn/support/download/linux-drivers.html，下文件`amdgpu-install_6.4.60402-1_all.deb`

2. `sudo dpkg -i amdgpu-install_6.4.60402-1_all.deb` 安装`amdgpu-install`脚本

3.  更新widnows驱动到[AMD Software: Adrenalin Edition™ 25.8.1 for WSL2](https://www.amd.com/en/resources/support-articles/release-notes/rn-rad-win-25-8-1.html).

4. 在这之前一定配置好国外的安装源，要下载很多文件，执行`amdgpu-install -y --usecase=wsl,rocm --no-dkms` 安装WSL usecase

5. 执行`rocminfo`查看版本信息

   ```bash
   *******
   Agent 1
   *******
     Name:                    AMD Ryzen 5 5600 6-Core Processor
     Uuid:                    CPU-XX
     Marketing Name:          AMD Ryzen 5 5600 6-Core Processor
     Vendor Name:             CPU
     Feature:                 None specified
     Profile:                 FULL_PROFILE
     Float Round Mode:        NEAR
   ```

   ​





#### 