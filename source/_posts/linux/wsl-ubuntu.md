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

### 安装过程

1. 安装WSL，打开系统设置-应用与功能-Windows 功能，勾选其中的`Virtual Machine Platform`和`Windows Subsystem for Linux`，重启电脑

2. 到[install-manual](https://learn.microsoft.com/en-us/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package) 下载[WSL2 Linux kernel update package for x64 machines](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)，并安装

3. PowerShell中执行`wsl --set-default-version 2`设置使用WSL2

4. ubuntu[官网](https://releases.ubuntu.com/noble/) 下载24.04 LTS的WSL的镜像文件[64-bit PC (AMD64) WSL image](https://releases.ubuntu.com/noble/ubuntu-24.04.3-wsl-amd64.wsl)，得到文件`ubuntu-24.04.3-wsl-amd64.wsl`

5. 把这个文件解压后得到1.3GB的`ubuntu-24.04.3-wsl-amd64`文件

6. 使用wsl导入系统镜像到指定目录`wsl --import <系统名称> <安装位置> <镜像文件路径>`
```powershell
wsl --import Ubuntu-24.04 "E:\wsl\Ubuntu-24.04" "E:\wsl\ubuntu-24.04.3-wsl-amd64"
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

#### 