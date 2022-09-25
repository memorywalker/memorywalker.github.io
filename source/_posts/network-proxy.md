---
title: Network Proxy
date: 2022-09-25 16:25:49
categories:
- network
tags:
- network
- clash
- proxy
---

## Network Proxy 

### Clash

参考 [Clash for Windows 优雅地使用 TUN 模式接管系统流量 | Dejavu's Blog](https://www.dejavu.moe/posts/cfw-tun/#:~:text=Clash for Windows 优雅地使用 TUN 模式接管系统流量 1 前言,，安装完成后 CFW 会自动重启 5 开启Mixin Mixin 开启 )

Clash目前是Windows上非常好用的代理软件，Android手机也有客户端，可以设置哪些应用走代理，规则设置自由。在Android TV上使用手机版本的Clash也很流畅，可以使用导入文件的方式导入代理，避免输入订阅地址。

![clash_setting](..\uploads\proxy\clash_setting.png)

#### 基本使用

* 导入订阅

  在Profiles界面输入框中输入订阅地址，点击下载后，就可以下载一个订阅到本地

* 代理服务

  代理服务的端口默认为7890端口

* 局域网共享代理

  如果需要给局域网中的其他网络设备，需要把**Allow LAN**选项打开，界面会提示当前共享服务的ip

* 全局HTTP代理

  如果需要代理整个系统的HTTP连接，需要把**System Proxy**选项打开，这样浏览器不用**Proxy SwitchyOmega**代理插件也可以使用代理



#### Tap代理

如果要给某个应用程序设置代理，而不只是浏览器的HTTP服务，可以使用Tap Service。

1. 点击Tap Service后面的管理安装Tap虚拟网卡

2. 安装成功后，在网络管理中可以看到一个`cfw-tap`的网络设备，此时是断开状态

3. 在Setting中，找到Mixin选项，选择YAML后，点击编辑输入以下代码

   ```yaml
   mixin: # object
     dns:
       enable: true    
       enhanced-mode: redir-host
       listen: :53
       nameserver:
         - https://doh.dns.sb/dns-query
         - https://dns.adguard.com/dns-query
         - https://cdn-doh.ssnm.xyz/dns-query
         - 119.29.29.29 #腾讯
         - 223.5.5.5 #阿里
   ```

4. 打开主界面的Mixin开关，此时cfw-tap网络就正常工作了

5. 打开**System Proxy**选项

6. 第三方的应用程序默认都会使用cfw-tap网络通信

#### TUN代理

1. 如果使用过tap模式，需要先把tap模式的网卡卸载

2. 点击Service Mode后的管理，安装服务模式，这个安装比较慢，等待安装成功后，小地球会变为绿色

3. 在Setting中找到Mixin，用YAML编辑以下内容

   ```yaml
   mixin: # Mixin 配置文件
     dns:
       enable: true 
       ipv6: true # true/false 是否启用 ipv6 支持 
       # 从 v0.18.8 版本开始，TUN 模式建议使用 fake-ip 模式，redir-host 将无法进行远端 DNS 解析
       enhanced-mode: fake-ip # redir-host/fake-ip
       # use-hosts: true # 查询 hosts 并返回 IP 记录
       default-nameserver: # 用于 DoH/DoT 的 Bootstrap Server
         - 223.5.5.5  # 阿里公共 DNS
         - 223.6.6.6  # 阿里公共 DNS
         - 119.29.29.29 # DNSPOD 公共 DNS
       fake-ip-range: 198.18.0.1/16 # Fake IP 地址池 (CIDR 形式)
       fake-ip-filter: # 微软系 APP 无法登陆使用等问题，通过添加 fake-ip-filter 解决
         # === Local ===
         - "*.lan"
         - "*.local"
         # === Microsoft Windows Serivice ===
         - "*.msftncsi.com"
         - "*.msftconnecttest.com"
       nameserver: # GeoIP 为 CN 时使用的 DNS NameServer（使用DoH/DoT）
         - https://doh.pub/dns-query   # DNSPod DoH
         - https://dns.alidns.com/dns-query  # 阿里 DoH
         #- https://[2400:3200::1]/dns-query  # 阿里 DoH
         #- https://[2400:3200:baba::1]/dns-query # 阿里 DoH
       fallback: # GeoIP 不是 CN 时使用的 DNS NameServer（使用DoH/DoT）
         #- https://doh.dns.sb/dns-query    # DNS.SB DoH
         - https://dns.google/dns-query    # Google DoH
         - https://1.1.1.1/dns-query   # Cloudflare DoH
         #- https://1.0.0.1/dns-query   # Cloudflare DoH
       fallback-filter:
         geoip: true # 启用 GeoIP
         ip-cidr:
           - 240.0.0.0/4
           - 127.0.0.1/8
           - 0.0.0.0/32
         domain:
           - +.google.com
           - +.facebook.com
           - +.twitter.com
           - +.youtube.com
           - +.xn--ngstr-lra8j.com
           - +.google.cn
           - +.googleapis.cn
           - +.googleapis.com
           - +.gvt1.com
   # interface-name: Ethernet  # 出口网卡名称（已注释），建议使用自动检测出口网卡模式👇
     tun:  # Tun 配置
       enable: true  # 启用 Tun 模式
   # 使用 system statck 需要 Clash Premium 2021.05.08 及更高版本
       stack: system  # gvisor/system 使用 system stack 请按照本文后面防火墙放行程序
       dns-hijack:   
         - 198.18.0.2:53  # 本地劫持 DNS 地址，无需修改
       auto-route: true
       auto-detect-interface: true  # 自动检测出口网卡
   rules: # 规则覆盖
     # 直连 IP 范围
     - IP-CIDR,0.0.0.0/8,DIRECT
     - IP-CIDR,10.0.0.0/8,DIRECT
     - IP-CIDR,100.64.0.0/10,DIRECT
     - IP-CIDR,127.0.0.0/8,DIRECT
     - IP-CIDR,169.254.0.0/16,DIRECT
     - IP-CIDR,172.16.0.0/12,DIRECT
     - IP-CIDR,192.0.0.0/24,DIRECT
     - IP-CIDR,192.0.2.0/24,DIRECT
     - IP-CIDR,192.88.99.0/24,DIRECT
     - IP-CIDR,192.168.0.0/16,DIRECT
     - IP-CIDR,198.18.0.0/15,DIRECT
     - IP-CIDR,198.51.100.0/24,DIRECT
     - IP-CIDR,203.0.113.0/24,DIRECT
     - IP-CIDR,223.255.255.0/24,DIRECT
     - IP-CIDR,224.0.0.0/4,DIRECT
     - IP-CIDR,240.0.0.0/4,DIRECT
     - IP-CIDR,255.255.255.255/32,DIRECT
     - IP-CIDR6,::/128,DIRECT
     - IP-CIDR6,::1/128,DIRECT
     - IP-CIDR6,100::/64,DIRECT
     - IP-CIDR6,64:ff9b::/96,DIRECT
     - IP-CIDR6,2001::/32,DIRECT
     - IP-CIDR6,2001:10::/28,DIRECT
     - IP-CIDR6,2001:20::/28,DIRECT
     - IP-CIDR6,2001:db8::/32,DIRECT
     - IP-CIDR6,2002::/16,DIRECT
     - IP-CIDR6,fc00::/7,DIRECT
     - IP-CIDR6,fe80::/10,DIRECT
     - IP-CIDR6,ff00::/8,DIRECT
   
     # Adguard 本地 DNS 请求直连
     - DOMAIN,injections.adguard.org,DIRECT
     - DOMAIN,local.adguard.org,DIRECT 
   
     # CN 网站全直连
     - DOMAIN-SUFFIX,cn,DIRECT
     - DOMAIN-KEYWORD,-cn,DIRECT
   
     - DOMAIN-SUFFIX,126.com,DIRECT
     - DOMAIN-SUFFIX,126.net,DIRECT
     - DOMAIN-SUFFIX,127.net,DIRECT
     - DOMAIN-SUFFIX,163.com,DIRECT
     - DOMAIN-SUFFIX,kugou.com,DIRECT
     - DOMAIN-SUFFIX,kuwo.cn,DIRECT
     - DOMAIN-SUFFIX,migu.cn,DIRECT
     - DOMAIN-SUFFIX,360buyimg.com,DIRECT
     - DOMAIN-SUFFIX,36kr.com,DIRECT
     - DOMAIN-SUFFIX,acfun.tv,DIRECT
     - DOMAIN-SUFFIX,air-matters.com,DIRECT
     - DOMAIN-SUFFIX,aixifan.com,DIRECT
     - DOMAIN-KEYWORD,alicdn,DIRECT
     - DOMAIN-KEYWORD,alipay,DIRECT
     - DOMAIN-KEYWORD,taobao,DIRECT
     - DOMAIN-SUFFIX,amap.com,DIRECT
     - DOMAIN-SUFFIX,autonavi.com,DIRECT
     - DOMAIN-KEYWORD,baidu,DIRECT
     - DOMAIN-SUFFIX,bdimg.com,DIRECT
     - DOMAIN-SUFFIX,bdstatic.com,DIRECT
     - DOMAIN-SUFFIX,bilibili.com,DIRECT
     - DOMAIN-SUFFIX,bilivideo.com,DIRECT
     - DOMAIN-SUFFIX,caiyunapp.com,DIRECT
     - DOMAIN-SUFFIX,clouddn.com,DIRECT
     - DOMAIN-SUFFIX,cnbeta.com,DIRECT
     - DOMAIN-SUFFIX,cnbetacdn.com,DIRECT
     - DOMAIN-SUFFIX,cootekservice.com,DIRECT
     - DOMAIN-SUFFIX,csdn.net,DIRECT
     - DOMAIN-SUFFIX,ctrip.com,DIRECT
     - DOMAIN-SUFFIX,dgtle.com,DIRECT
     - DOMAIN-SUFFIX,dianping.com,DIRECT
     - DOMAIN-SUFFIX,douban.com,DIRECT
     - DOMAIN-SUFFIX,doubanio.com,DIRECT
     - DOMAIN-SUFFIX,duokan.com,DIRECT
     - DOMAIN-SUFFIX,easou.com,DIRECT
     - DOMAIN-SUFFIX,ele.me,DIRECT
     - DOMAIN-SUFFIX,feng.com,DIRECT
     - DOMAIN-SUFFIX,fir.im,DIRECT
     - DOMAIN-SUFFIX,frdic.com,DIRECT
     - DOMAIN-SUFFIX,g-cores.com,DIRECT
     - DOMAIN-SUFFIX,godic.net,DIRECT
     - DOMAIN-SUFFIX,gtimg.com,DIRECT
     - DOMAIN,cdn.hockeyapp.net,DIRECT
     - DOMAIN-SUFFIX,hongxiu.com,DIRECT
     - DOMAIN-SUFFIX,hxcdn.net,DIRECT
     - DOMAIN-SUFFIX,iciba.com,DIRECT
     - DOMAIN-SUFFIX,ifeng.com,DIRECT
     - DOMAIN-SUFFIX,ifengimg.com,DIRECT
     - DOMAIN-SUFFIX,ipip.net,DIRECT
     - DOMAIN-SUFFIX,iqiyi.com,DIRECT
     - DOMAIN-SUFFIX,jd.com,DIRECT
     - DOMAIN-SUFFIX,jianshu.com,DIRECT
     - DOMAIN-SUFFIX,knewone.com,DIRECT
     - DOMAIN-SUFFIX,le.com,DIRECT
     - DOMAIN-SUFFIX,lecloud.com,DIRECT
     - DOMAIN-SUFFIX,lemicp.com,DIRECT
     - DOMAIN-SUFFIX,licdn.com,DIRECT
     - DOMAIN-SUFFIX,linkedin.com,DIRECT
     - DOMAIN-SUFFIX,luoo.net,DIRECT
     - DOMAIN-SUFFIX,meituan.com,DIRECT
     - DOMAIN-SUFFIX,meituan.net,DIRECT
     - DOMAIN-SUFFIX,mi.com,DIRECT
     - DOMAIN-SUFFIX,miaopai.com,DIRECT
     - DOMAIN-SUFFIX,microsoft.com,DIRECT
     - DOMAIN-SUFFIX,microsoftonline.com,DIRECT
     - DOMAIN-SUFFIX,miui.com,DIRECT
     - DOMAIN-SUFFIX,miwifi.com,DIRECT
     - DOMAIN-SUFFIX,mob.com,DIRECT
     - DOMAIN-SUFFIX,netease.com,DIRECT
     - DOMAIN-SUFFIX,office.com,DIRECT
     - DOMAIN-SUFFIX,office365.com,DIRECT
     - DOMAIN-KEYWORD,officecdn,DIRECT
     - DOMAIN-SUFFIX,oschina.net,DIRECT
     - DOMAIN-SUFFIX,ppsimg.com,DIRECT
     - DOMAIN-SUFFIX,pstatp.com,DIRECT
     - DOMAIN-SUFFIX,qcloud.com,DIRECT
     - DOMAIN-SUFFIX,qdaily.com,DIRECT
     - DOMAIN-SUFFIX,qdmm.com,DIRECT
     - DOMAIN-SUFFIX,qhimg.com,DIRECT
     - DOMAIN-SUFFIX,qhres.com,DIRECT
     - DOMAIN-SUFFIX,qidian.com,DIRECT
     - DOMAIN-SUFFIX,qihucdn.com,DIRECT
     - DOMAIN-SUFFIX,qiniu.com,DIRECT
     - DOMAIN-SUFFIX,qiniucdn.com,DIRECT
     - DOMAIN-SUFFIX,qiyipic.com,DIRECT
     - DOMAIN-SUFFIX,qq.com,DIRECT
     - DOMAIN-SUFFIX,qqurl.com,DIRECT
     - DOMAIN-SUFFIX,rarbg.to,DIRECT
     - DOMAIN-SUFFIX,ruguoapp.com,DIRECT
     - DOMAIN-SUFFIX,segmentfault.com,DIRECT
     - DOMAIN-SUFFIX,sinaapp.com,DIRECT
     - DOMAIN-SUFFIX,smzdm.com,DIRECT
     - DOMAIN-SUFFIX,snapdrop.net,DIRECT
     - DOMAIN-SUFFIX,sogou.com,DIRECT
     - DOMAIN-SUFFIX,sogoucdn.com,DIRECT
     - DOMAIN-SUFFIX,sohu.com,DIRECT
     - DOMAIN-SUFFIX,soku.com,DIRECT
     - DOMAIN-SUFFIX,speedtest.net,DIRECT
     - DOMAIN-SUFFIX,sspai.com,DIRECT
     - DOMAIN-SUFFIX,suning.com,DIRECT
     - DOMAIN-SUFFIX,taobao.com,DIRECT
     - DOMAIN-SUFFIX,tencent.com,DIRECT
     - DOMAIN-SUFFIX,tenpay.com,DIRECT
     - DOMAIN-SUFFIX,tianyancha.com,DIRECT
     - DOMAIN-SUFFIX,tmall.com,DIRECT
     - DOMAIN-SUFFIX,tudou.com,DIRECT
     - DOMAIN-SUFFIX,umetrip.com,DIRECT
     - DOMAIN-SUFFIX,upaiyun.com,DIRECT
     - DOMAIN-SUFFIX,upyun.com,DIRECT
     - DOMAIN-SUFFIX,veryzhun.com,DIRECT
     - DOMAIN-SUFFIX,weather.com,DIRECT
     - DOMAIN-SUFFIX,weibo.com,DIRECT
     - DOMAIN-SUFFIX,xiami.com,DIRECT
     - DOMAIN-SUFFIX,xiami.net,DIRECT
     - DOMAIN-SUFFIX,xiaomicp.com,DIRECT
     - DOMAIN-SUFFIX,ximalaya.com,DIRECT
     - DOMAIN-SUFFIX,xmcdn.com,DIRECT
     - DOMAIN-SUFFIX,xunlei.com,DIRECT
     - DOMAIN-SUFFIX,yhd.com,DIRECT
     - DOMAIN-SUFFIX,yihaodianimg.com,DIRECT
     - DOMAIN-SUFFIX,yinxiang.com,DIRECT
     - DOMAIN-SUFFIX,ykimg.com,DIRECT
     - DOMAIN-SUFFIX,youdao.com,DIRECT
     - DOMAIN-SUFFIX,youku.com,DIRECT
     - DOMAIN-SUFFIX,zealer.com,DIRECT
     - DOMAIN-SUFFIX,zhihu.com,DIRECT
     - DOMAIN-SUFFIX,zhimg.com,DIRECT
     - DOMAIN-SUFFIX,zimuzu.tv,DIRECT
     - DOMAIN-SUFFIX,zoho.com,DIRECT
   
   
    # Telegram 相关全代理
     - DOMAIN-SUFFIX,telegra.ph,Proxy
     - DOMAIN-SUFFIX,telegram.org,Proxy
     - IP-CIDR,91.108.4.0/22,Proxy
     - IP-CIDR,91.108.8.0/21,Proxy
     - IP-CIDR,91.108.16.0/22,Proxy
     - IP-CIDR,91.108.56.0/22,Proxy
     - IP-CIDR,149.154.160.0/20,Proxy
     - IP-CIDR6,2001:67c:4e8::/48,Proxy
     - IP-CIDR6,2001:b28:f23d::/48,Proxy
     - IP-CIDR6,2001:b28:f23f::/48,Proxy
   
   # 海外网站
     - DOMAIN-SUFFIX,9to5mac.com,Proxy
     - DOMAIN-SUFFIX,abpchina.org,Proxy
     - DOMAIN-SUFFIX,adblockplus.org,Proxy
     - DOMAIN-SUFFIX,adobe.com,Proxy
     - DOMAIN-SUFFIX,akamaized.net,Proxy
     - DOMAIN-SUFFIX,alfredapp.com,Proxy
     - DOMAIN-SUFFIX,amplitude.com,Proxy
     - DOMAIN-SUFFIX,ampproject.org,Proxy
     - DOMAIN-SUFFIX,android.com,Proxy
     - DOMAIN-SUFFIX,angularjs.org,Proxy
     - DOMAIN-SUFFIX,aolcdn.com,Proxy
     - DOMAIN-SUFFIX,apkpure.com,Proxy
     - DOMAIN-SUFFIX,appledaily.com,Proxy
     - DOMAIN-SUFFIX,appshopper.com,Proxy
     - DOMAIN-SUFFIX,appspot.com,Proxy
     - DOMAIN-SUFFIX,arcgis.com,Proxy
     - DOMAIN-SUFFIX,archive.org,Proxy
     - DOMAIN-SUFFIX,armorgames.com,Proxy
     - DOMAIN-SUFFIX,aspnetcdn.com,Proxy
     - DOMAIN-SUFFIX,att.com,Proxy
     - DOMAIN-SUFFIX,awsstatic.com,Proxy
     - DOMAIN-SUFFIX,azureedge.net,Proxy
     - DOMAIN-SUFFIX,azurewebsites.net,Proxy
     - DOMAIN-SUFFIX,bing.com,Proxy
     - DOMAIN-SUFFIX,bintray.com,Proxy
     - DOMAIN-SUFFIX,bit.com,Proxy
     - DOMAIN-SUFFIX,bit.ly,Proxy
     - DOMAIN-SUFFIX,bitbucket.org,Proxy
     - DOMAIN-SUFFIX,bjango.com,Proxy
     - DOMAIN-SUFFIX,bkrtx.com,Proxy
     - DOMAIN-SUFFIX,blog.com,Proxy
     - DOMAIN-SUFFIX,blogcdn.com,Proxy
     - DOMAIN-SUFFIX,blogger.com,Proxy
     - DOMAIN-SUFFIX,blogsmithmedia.com,Proxy
     - DOMAIN-SUFFIX,blogspot.com,Proxy
     - DOMAIN-SUFFIX,blogspot.hk,Proxy
     - DOMAIN-SUFFIX,bloomberg.com,Proxy
     - DOMAIN-SUFFIX,box.com,Proxy
     - DOMAIN-SUFFIX,box.net,Proxy
     - DOMAIN-SUFFIX,cachefly.net,Proxy
     - DOMAIN-SUFFIX,chromium.org,Proxy
     - DOMAIN-SUFFIX,cl.ly,Proxy
     - DOMAIN-SUFFIX,cloudflare.com,Proxy
     - DOMAIN-SUFFIX,cloudfront.net,Proxy
     - DOMAIN-SUFFIX,cloudmagic.com,Proxy
     - DOMAIN-SUFFIX,cmail19.com,Proxy
     - DOMAIN-SUFFIX,cnet.com,Proxy
     - DOMAIN-SUFFIX,cocoapods.org,Proxy
     - DOMAIN-SUFFIX,comodoca.com,Proxy
     - DOMAIN-SUFFIX,crashlytics.com,Proxy
     - DOMAIN-SUFFIX,culturedcode.com,Proxy
     - DOMAIN-SUFFIX,d.pr,Proxy
     - DOMAIN-SUFFIX,danilo.to,Proxy
     - DOMAIN-SUFFIX,dayone.me,Proxy
     - DOMAIN-SUFFIX,db.tt,Proxy
     - DOMAIN-SUFFIX,deskconnect.com,Proxy
     - DOMAIN-SUFFIX,disq.us,Proxy
     - DOMAIN-SUFFIX,disqus.com,Proxy
     - DOMAIN-SUFFIX,disquscdn.com,Proxy
     - DOMAIN-SUFFIX,dnsimple.com,Proxy
     - DOMAIN-SUFFIX,docker.com,Proxy
     - DOMAIN-SUFFIX,dribbble.com,Proxy
     - DOMAIN-SUFFIX,droplr.com,Proxy
     - DOMAIN-SUFFIX,duckduckgo.com,Proxy
     - DOMAIN-SUFFIX,dueapp.com,Proxy
     - DOMAIN-SUFFIX,dytt8.net,Proxy
     - DOMAIN-SUFFIX,edgecastcdn.net,Proxy
     - DOMAIN-SUFFIX,edgekey.net,Proxy
     - DOMAIN-SUFFIX,edgesuite.net,Proxy
     - DOMAIN-SUFFIX,engadget.com,Proxy
     - DOMAIN-SUFFIX,entrust.net,Proxy
     - DOMAIN-SUFFIX,eurekavpt.com,Proxy
     - DOMAIN-SUFFIX,evernote.com,Proxy
     - DOMAIN-SUFFIX,fabric.io,Proxy
     - DOMAIN-SUFFIX,fast.com,Proxy
     - DOMAIN-SUFFIX,fastly.net,Proxy
     - DOMAIN-SUFFIX,fc2.com,Proxy
     - DOMAIN-SUFFIX,feedburner.com,Proxy
     - DOMAIN-SUFFIX,feedly.com,Proxy
     - DOMAIN-SUFFIX,feedsportal.com,Proxy
     - DOMAIN-SUFFIX,fiftythree.com,Proxy
     - DOMAIN-SUFFIX,firebaseio.com,Proxy
     - DOMAIN-SUFFIX,flexibits.com,Proxy
     - DOMAIN-SUFFIX,flickr.com,Proxy
     - DOMAIN-SUFFIX,flipboard.com,Proxy
     - DOMAIN-SUFFIX,g.co,Proxy
     - DOMAIN-SUFFIX,gabia.net,Proxy
     - DOMAIN-SUFFIX,geni.us,Proxy
     - DOMAIN-SUFFIX,gfx.ms,Proxy
     - DOMAIN-SUFFIX,ggpht.com,Proxy
     - DOMAIN-SUFFIX,ghostnoteapp.com,Proxy
     - DOMAIN-SUFFIX,git.io,Proxy
     - DOMAIN-KEYWORD,github,Proxy
     - DOMAIN-SUFFIX,globalsign.com,Proxy
     - DOMAIN-SUFFIX,gmodules.com,Proxy
     - DOMAIN-SUFFIX,godaddy.com,Proxy
     - DOMAIN-SUFFIX,golang.org,Proxy
     - DOMAIN-SUFFIX,gongm.in,Proxy
     - DOMAIN-SUFFIX,goo.gl,Proxy
     - DOMAIN-SUFFIX,goodreaders.com,Proxy
     - DOMAIN-SUFFIX,goodreads.com,Proxy
     - DOMAIN-SUFFIX,gravatar.com,Proxy
     - DOMAIN-SUFFIX,gstatic.com,Proxy
     - DOMAIN-SUFFIX,gvt0.com,Proxy
     - DOMAIN-SUFFIX,hockeyapp.net,Proxy
     - DOMAIN-SUFFIX,hotmail.com,Proxy
     - DOMAIN-SUFFIX,icons8.com,Proxy
     - DOMAIN-SUFFIX,ifixit.com,Proxy
     - DOMAIN-SUFFIX,ift.tt,Proxy
     - DOMAIN-SUFFIX,ifttt.com,Proxy
     - DOMAIN-SUFFIX,iherb.com,Proxy
     - DOMAIN-SUFFIX,imageshack.us,Proxy
     - DOMAIN-SUFFIX,img.ly,Proxy
     - DOMAIN-SUFFIX,imgur.com,Proxy
     - DOMAIN-SUFFIX,imore.com,Proxy
     - DOMAIN-SUFFIX,instapaper.com,Proxy
     - DOMAIN-SUFFIX,ipn.li,Proxy
     - DOMAIN-SUFFIX,is.gd,Proxy
     - DOMAIN-SUFFIX,issuu.com,Proxy
     - DOMAIN-SUFFIX,itgonglun.com,Proxy
     - DOMAIN-SUFFIX,itun.es,Proxy
     - DOMAIN-SUFFIX,ixquick.com,Proxy
     - DOMAIN-SUFFIX,j.mp,Proxy
     - DOMAIN-SUFFIX,js.revsci.net,Proxy
     - DOMAIN-SUFFIX,jshint.com,Proxy
     - DOMAIN-SUFFIX,jtvnw.net,Proxy
     - DOMAIN-SUFFIX,justgetflux.com,Proxy
     - DOMAIN-SUFFIX,kat.cr,Proxy
     - DOMAIN-SUFFIX,klip.me,Proxy
     - DOMAIN-SUFFIX,libsyn.com,Proxy
     - DOMAIN-SUFFIX,linode.com,Proxy
     - DOMAIN-SUFFIX,lithium.com,Proxy
     - DOMAIN-SUFFIX,littlehj.com,Proxy
     - DOMAIN-SUFFIX,live.com,Proxy
     - DOMAIN-SUFFIX,live.net,Proxy
     - DOMAIN-SUFFIX,livefilestore.com,Proxy
     - DOMAIN-SUFFIX,llnwd.net,Proxy
     - DOMAIN-SUFFIX,macid.co,Proxy
     - DOMAIN-SUFFIX,macromedia.com,Proxy
     - DOMAIN-SUFFIX,macrumors.com,Proxy
     - DOMAIN-SUFFIX,mashable.com,Proxy
     - DOMAIN-SUFFIX,mathjax.org,Proxy
     - DOMAIN-SUFFIX,medium.com,Proxy
     - DOMAIN-SUFFIX,mega.co.nz,Proxy
     - DOMAIN-SUFFIX,mega.nz,Proxy
     - DOMAIN-SUFFIX,megaupload.com,Proxy
     - DOMAIN-SUFFIX,microsofttranslator.com,Proxy
     - DOMAIN-SUFFIX,mindnode.com,Proxy
     - DOMAIN-SUFFIX,mobile01.com,Proxy
     - DOMAIN-SUFFIX,modmyi.com,Proxy
     - DOMAIN-SUFFIX,msedge.net,Proxy
     - DOMAIN-SUFFIX,myfontastic.com,Proxy
     - DOMAIN-SUFFIX,name.com,Proxy
     - DOMAIN-SUFFIX,nextmedia.com,Proxy
     - DOMAIN-SUFFIX,nsstatic.net,Proxy
     - DOMAIN-SUFFIX,nssurge.com,Proxy
     - DOMAIN-SUFFIX,nyt.com,Proxy
     - DOMAIN-SUFFIX,nytimes.com,Proxy
     - DOMAIN-SUFFIX,omnigroup.com,Proxy
     - DOMAIN-SUFFIX,onedrive.com,Proxy
     - DOMAIN-SUFFIX,onenote.com,Proxy
     - DOMAIN-SUFFIX,ooyala.com,Proxy
     - DOMAIN-SUFFIX,openvpn.net,Proxy
     - DOMAIN-SUFFIX,openwrt.org,Proxy
     - DOMAIN-SUFFIX,orkut.com,Proxy
     - DOMAIN-SUFFIX,osxdaily.com,Proxy
     - DOMAIN-SUFFIX,outlook.com,Proxy
     - DOMAIN-SUFFIX,ow.ly,Proxy
     - DOMAIN-SUFFIX,paddleapi.com,Proxy
     - DOMAIN-SUFFIX,parallels.com,Proxy
     - DOMAIN-SUFFIX,parse.com,Proxy
     - DOMAIN-SUFFIX,pdfexpert.com,Proxy
     - DOMAIN-SUFFIX,periscope.tv,Proxy
     - DOMAIN-SUFFIX,pinboard.in,Proxy
     - DOMAIN-SUFFIX,pinterest.com,Proxy
     - DOMAIN-SUFFIX,pixelmator.com,Proxy
     - DOMAIN-SUFFIX,pixiv.net,Proxy
     - DOMAIN-SUFFIX,playpcesor.com,Proxy
     - DOMAIN-SUFFIX,playstation.com,Proxy
     - DOMAIN-SUFFIX,playstation.com.hk,Proxy
     - DOMAIN-SUFFIX,playstation.net,Proxy
     - DOMAIN-SUFFIX,playstationnetwork.com,Proxy
     - DOMAIN-SUFFIX,pushwoosh.com,Proxy
     - DOMAIN-SUFFIX,rime.im,Proxy
     - DOMAIN-SUFFIX,servebom.com,Proxy
     - DOMAIN-SUFFIX,sfx.ms,Proxy
     - DOMAIN-SUFFIX,shadowsocks.org,Proxy
     - DOMAIN-SUFFIX,sharethis.com,Proxy
     - DOMAIN-SUFFIX,shazam.com,Proxy
     - DOMAIN-SUFFIX,skype.com,Proxy
     - DOMAIN-SUFFIX,smartdnsProxy.com,Proxy
     - DOMAIN-SUFFIX,smartmailcloud.com,Proxy
     - DOMAIN-SUFFIX,sndcdn.com,Proxy
     - DOMAIN-SUFFIX,sony.com,Proxy
     - DOMAIN-SUFFIX,soundcloud.com,Proxy
     - DOMAIN-SUFFIX,sourceforge.net,Proxy
     - DOMAIN-SUFFIX,spotify.com,Proxy
     - DOMAIN-SUFFIX,squarespace.com,Proxy
     - DOMAIN-SUFFIX,sstatic.net,Proxy
     - DOMAIN-SUFFIX,st.luluku.pw,Proxy
     - DOMAIN-SUFFIX,stackoverflow.com,Proxy
     - DOMAIN-SUFFIX,startpage.com,Proxy
     - DOMAIN-SUFFIX,staticflickr.com,Proxy
     - DOMAIN-SUFFIX,steamcommunity.com,Proxy
     - DOMAIN-SUFFIX,symauth.com,Proxy
     - DOMAIN-SUFFIX,symcb.com,Proxy
     - DOMAIN-SUFFIX,symcd.com,Proxy
     - DOMAIN-SUFFIX,tapbots.com,Proxy
     - DOMAIN-SUFFIX,tapbots.net,Proxy
     - DOMAIN-SUFFIX,tdesktop.com,Proxy
     - DOMAIN-SUFFIX,techcrunch.com,Proxy
     - DOMAIN-SUFFIX,techsmith.com,Proxy
     - DOMAIN-SUFFIX,thepiratebay.org,Proxy
     - DOMAIN-SUFFIX,theverge.com,Proxy
     - DOMAIN-SUFFIX,time.com,Proxy
     - DOMAIN-SUFFIX,timeinc.net,Proxy
     - DOMAIN-SUFFIX,tiny.cc,Proxy
     - DOMAIN-SUFFIX,tinypic.com,Proxy
     - DOMAIN-SUFFIX,tmblr.co,Proxy
     - DOMAIN-SUFFIX,todoist.com,Proxy
     - DOMAIN-SUFFIX,trello.com,Proxy
     - DOMAIN-SUFFIX,trustasiassl.com,Proxy
     - DOMAIN-SUFFIX,tumblr.co,Proxy
     - DOMAIN-SUFFIX,tumblr.com,Proxy
     - DOMAIN-SUFFIX,tweetdeck.com,Proxy
     - DOMAIN-SUFFIX,tweetmarker.net,Proxy
     - DOMAIN-SUFFIX,twitch.tv,Proxy
     - DOMAIN-SUFFIX,txmblr.com,Proxy
     - DOMAIN-SUFFIX,typekit.net,Proxy
     - DOMAIN-SUFFIX,ubertags.com,Proxy
     - DOMAIN-SUFFIX,ublock.org,Proxy
     - DOMAIN-SUFFIX,ubnt.com,Proxy
     - DOMAIN-SUFFIX,ulyssesapp.com,Proxy
     - DOMAIN-SUFFIX,urchin.com,Proxy
     - DOMAIN-SUFFIX,usertrust.com,Proxy
     - DOMAIN-SUFFIX,v.gd,Proxy
     - DOMAIN-SUFFIX,v2ex.com,Proxy
     - DOMAIN-SUFFIX,vimeo.com,Proxy
     - DOMAIN-SUFFIX,vimeocdn.com,Proxy
     - DOMAIN-SUFFIX,vine.co,Proxy
     - DOMAIN-SUFFIX,vivaldi.com,Proxy
     - DOMAIN-SUFFIX,vox-cdn.com,Proxy
     - DOMAIN-SUFFIX,vsco.co,Proxy
     - DOMAIN-SUFFIX,vultr.com,Proxy
     - DOMAIN-SUFFIX,w.org,Proxy
     - DOMAIN-SUFFIX,w3schools.com,Proxy
     - DOMAIN-SUFFIX,webtype.com,Proxy
     - DOMAIN-SUFFIX,wikiwand.com,Proxy
     - DOMAIN-SUFFIX,wikileaks.org,Proxy
     - DOMAIN-SUFFIX,wikimedia.org,Proxy
     - DOMAIN-SUFFIX,wikipedia.com,Proxy
     - DOMAIN-SUFFIX,wikipedia.org,Proxy
     - DOMAIN-SUFFIX,windows.com,Proxy
     - DOMAIN-SUFFIX,windows.net,Proxy
     - DOMAIN-SUFFIX,wire.com,Proxy
     - DOMAIN-SUFFIX,wordpress.com,Proxy
     - DOMAIN-SUFFIX,workflowy.com,Proxy
     - DOMAIN-SUFFIX,wp.com,Proxy
     - DOMAIN-SUFFIX,wsj.com,Proxy
     - DOMAIN-SUFFIX,wsj.net,Proxy
     - DOMAIN-SUFFIX,xda-developers.com,Proxy
     - DOMAIN-SUFFIX,xeeno.com,Proxy
     - DOMAIN-SUFFIX,xiti.com,Proxy
     - DOMAIN-SUFFIX,yahoo.com,Proxy
     - DOMAIN-SUFFIX,yimg.com,Proxy
     - DOMAIN-SUFFIX,ying.com,Proxy
     - DOMAIN-SUFFIX,yoyo.org,Proxy
     - DOMAIN-SUFFIX,ytimg.com,Proxy
   
     # 最终规则
     - GEOIP,CN,DIRECT
     - MATCH,PROXY
   
   ```

4. 打开Mixin选项

5. 关闭System Proxy选项

6. 系统中会多一个名称为Clash的虚拟网卡，网络流量走这个网卡



