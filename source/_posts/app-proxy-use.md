---
title: 应用程序网络代理
date: 2020-02-23 20:25:49
tags: network; proxifier
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



### SocksCap64使用

### SSTAP使用







