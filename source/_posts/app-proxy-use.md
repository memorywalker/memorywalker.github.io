---
title: 应用程序网络代理
date: 2020-02-23 20:25:49
categories:
- network
tags:
- network
- proxifier
---

### Proxifier使用

启动SSR之后，**不用**选择服务器负载均衡，系统代理模式选择**直连**或**PAC**都可以

1. 设置服务器

   使用默认的127.0.0.1端口为1080

   ![proxifier_server](/uploads/proxy/proxifier_server.png)

2. 设置域名解析

   不设置也可以，如果域名解析失败需要通过代理解析再设置

   ![proxifier_dns](/uploads/proxy/proxifier_dns.png)

3. 设置代理规则

   可以设置对一个程序禁止访问一些目标网址，action选择block

   可以设置全局所有程序都走proxifier，application保留any不变，action选择刚刚的服务器，同时由于不能让SSR也走proxifier，所以需要新建一个rule，让ssr走direct即可

   ![proxifier_rules](/uploads/proxy/proxifier_rules.png)   

4. 运行程序后，显示数据包转发过程

   epic客户端使用

   ![proxifier_using](/uploads/proxy/proxifier_using.png)



#### 游戏加速

玩GTA5的线上模式时，每日的赌场任务如果是裸连或香港的IP，无法游玩大转盘，虽然用联通手机开热点可以直接连接线上模式

在[keylol论坛](https://keylol.com/t290826-1-1)看到分享的GTA5代理设置，试了一下用美区代理可以玩转盘了，网络还还是挺稳定的。每次保存战局中的内容时会触发网络连接。

新增3个代理规则：

* GTA加速

  应用程序: subprocess.exe; gta5.exe; gtavlauncher.exe;

  目标主机：

  ```
  conductor-prod.ros.rockstargames.com; 
  auth-prod.ros.rockstargames.com; 
  prod.cloud.rockstargames.com; 
  ```

  动作：选择配置好的sock5代理服务

* GTA分析禁连

  应用程序: subprocess.exe; gta5.exe; gtavlauncher.exe;

  目标主机：

  ```
  www.google-analytics.com;
  stats.g.doubleclick.net;
  www.google.com;
  ```

  动作：Block

* GTA识别

  应用程序: gta5.exe; gtavlauncher.exe;

  目标主机：prod.ros.rockstargames.com;

  动作：选择配置好的sock5代理服务

游戏运行过程中会在状态窗口中刷

```shell
[03.07 19:49:28] GTA5.exe *64 - prod.p02sjc.pod.rockstargames.com:443 打开通过代理 127.0.0.1:10808 SOCKS5
[03.07 19:49:30] GTA5.exe *64 - prod.p02sjc.pod.rockstargames.com:443 关闭，965 字节已发送，5005 字节 (4.88 KB) 已接收，生存期 00:02
[03.07 19:49:51] GTA5.exe *64 - prod.ros.rockstargames.com:80 打开通过代理 127.0.0.1:10808 SOCKS5
[03.07 19:49:54] GTA5.exe *64 - prod.ros.rockstargames.com:80 关闭，643 字节已发送，13001 字节 (12.6 KB) 已接收，生存期 00:03
```

##### GTA5 相关备注

* 完成全福银行任务后，可以用批发价买骷髅马装甲版，这个车必须买，之后可以在车里做R星制作的任务刷等级和钱
* 北京时间每周四晚更新每周的活动，每周的活动有物品打折和新的玩法，赌场更新汽车奖品
* 有钱后可以先买公寓20W的，通过观光客任务一次2.5W，每次用时15分钟
* 可以创建两个角色，两个角色银行共享，其他都不共享，资产都要各自买，R星的奖励左轮枪任务、寻宝任务和帐号绑定，只能领取一次

### SocksCap64使用

### SSTAP使用







