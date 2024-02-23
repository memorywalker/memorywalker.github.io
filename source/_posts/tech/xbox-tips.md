---
title: Xbox Tips
date: 2022-07-11 23:25:49
categories:
- game
tags:
- game
- xbox
- proxy
---

## Xbox 

### Xbox Proxy

xbox的Rewards需要美区IP才能激活，同时部分游戏也需要加速器才能正常联机使用，自己实际使用的时间不多，也没必要购买各种加速器。由于Xbox不支持设置代理功能，因此如果没有刷机的路由器或加速盒子，就只能开一个电脑进行转发。在网上搜了一下找到一个开源项目

#### **pcap2socks**

https://github.com/zhxie/pcap2socks

它还有个前端UI界面工程，[pcap2socks GUI](https://github.com/zhxie/pcap2socks-gui) 实际上使用命令行已经足够了。

##### 使用方法

1. 开启自己的Clash软件，设置全局加速
2. 下载`pcap2socks.exe`，把它放在一个英文目录中
3. 在目录中新建一个bat脚本`proxy_xbox.bat`，内容为`pcap2socks -s 172.2.2.2 -p 172.2.2.1 -d 127.0.0.1:7890 -i "\Device\NPF_{6DADC48E-B6C8-4920-9B93-3BBCF597A8D5}"`
4. 运行批处理后，会提示当前代理的IP地址，网关和掩码，并等待连接`Proxy 172.2.2.2/32 to 127.0.0.1:7890`
5. 在xbox的网络设置中，进阶设置中，设置有线网的IP地址为手动，将代理的IP`172.2.2.2`，掩码`255.255.255.0`以及网关`172.2.2.1`输入设置，DNS设置一个自己路由器的默认网关例如`192.168.68.1`和一个备用DNS地址`8.8.8.8`
6. 如果Clash代理没有问题的话，Xbox就可以使用代理进行连接了



##### 命令说明

`pcap2socks -s <需要代理的设备的 IP 地址> -p <需要代理的设备上所填写的网关> -d <SOCKS 代理，如 127.0.0.1:1080> -i <网卡名称>`

其中如果电脑有多个网卡，需要指定网卡，如果不设置`-i`参数，会提示`error: Cannot determine the interface. Available interfaces are listed below`，可以从程序输出的列表中查看自己是哪个网卡的ip地址和xbox的在同一个局域网中，使用那个网卡的名称作为参数。例如我本机的输出中最后一个无线网卡和xbox在同一个局域网，所以配置的网卡参数为`"\Device\NPF_{6DADC48E-B6C8-4920-9B93-3BBCF597A8D5}"`：

```powershell
D:\network\pcap2socks-v0.6.2-windows-amd64>pcap2socks -s 172.2.2.2 -p 172.2.2.1
-d 127.0.0.1:7890
Interface '{7667A9BA-BB55-4645-B68B-771977DC791E}' has an unexpected address len
gth: 8
Interface '{EF09116C-B70F-479F-96B5-985556028D0F}' has an unexpected address len
gth: 8
error: Cannot determine the interface. Available interfaces are listed below, an
d please use -i <INTERFACE> to designate:
Interface '{7667A9BA-BB55-4645-B68B-771977DC791E}' has an unexpected address len
gth: 8
Interface '{EF09116C-B70F-479F-96B5-985556028D0F}' has an unexpected address len
gth: 8
    \Device\NPF_{04D21285-A380-4EE3-BA6F-BA624E1AE318} (VirtualBox Host-Only Eth
ernet Adapter) [0a:00:37:00:00:28]: 192.168.56.1
    \Device\NPF_{6DADC48E-B6C8-4920-9B93-3BBCF597A8D5} (Atheros AR9285 Wireless
Network Adapter) [6c:fd:b7:33:78:ad]: 10.1.1.151
```



