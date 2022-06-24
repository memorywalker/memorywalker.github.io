---
title: Steam ASF
date: 2021-01-05 20:30:49
categories:
- game
tags:
- steam
- asf
---

## ASF
ArchiSteamFarm 

[官方文档]: https://github.com/JustArchiNET/ArchiSteamFarm/wiki/Setting-up

这是一个服务器端的程序，当然也可以在本地的PC上运行

1. 可以用来挂卡
2. 和小号聊天，让机器人执行命令，批量激活游戏 

### Install
#### Install .NET Core prerequisites

* Microsoft Visual C++ 2015 Redistributable Update 3 RC
* KB2533623 and KB2999226

For Linux:

* libcurl3 (libcurl)
* libicu60 (libicu, latest version for your distribution, for example libicu57 for Debian 9)
* libkrb5-3 (krb5-libs)
* liblttng-ust0 (lttng-ust)
* libssl1.0.2 (libssl, openssl-libs, latest 1.0.X version for your distribution)
* zlib1g (zlib)

#### Download latest ASF release
From [here](https://github.com/JustArchi/ArchiSteamFarm/releases/latest)

windows 64位 下载这个**ASF-win-x64**

recommend file structrue

	C:\ASF (where you put your own things)
	    ├── ASF shortcut.lnk (optional)
	    ├── Config shortcut.lnk (optional)
	    ├── Commands.txt (optional)
	    ├── MyExtraScript.bat (optional)
	    ├── ... (any other files of your choice, optional)
	    └── Core (dedicated to ASF only, where you extract the archive)
	         ├── ArchiSteamFarm.dll
	         ├── config
	         └── (...)

#### Configure ASF
##### Web 配置

* 可以直接到官方提供的网站配置，这个网页只是客户端执行，因此不要担心帐号被盗[here](https://justarchi.github.io/ArchiSteamFarm/#/)
* 也可以把那个网页下载下来，在本地浏览器打开，这个工具只是js写的，不需要服务器环境
* 直接拷贝模板配置，修改配置文件

切换到Bot选项：
1. 输入一个Bot的名字，不能是`ASF`，`example`以及`minimal`，因为默认配置目录已经有了这3个文件
2. steam的用户名和密码这里如果不填，每次启动asf时，需要与程序交互输入密码，如果是本地使用建议填上密码，也可以生成配置文件后手动增加的配置文件中
3. 勾选Enabled
4. 点击下载json格式的配置文件，并把这个文件放入config目录

#### Launch ASF 
点击ArchiSteamFarm.exe启动asf，第一次登录过程中，需要输入steam guard

如果steam的帐号解锁了5美元限制，系统会自动挂卡，并显示每个游戏还有多少个卡

![limited](/uploads/steam/asf_account_limited.png)

#### Extended configuration

* ASF支持同时挂多个帐号，只需要将帐号的配置文件放到config目录即可，一个帐号配置如`tip_bot.json`

  ```json
  {
    "SteamLogin": "loginname",
    "SteamPassword": "password",
    "Enabled": true,
    "AutoSteamSaleEvent": true,
    "SteamUserPermissions": {
      "76561199116482158": 3
    }
  }
  ```

  

* 可以自定义设置挂卡时显示的游戏信息，在配置页面的高级选项中，编辑`CustomGamePlayedWhileFarming`为你想显示的文字，这样看不到当前正在挂哪个游戏。

* 配置页面的ASF选项页是针对ASF的全局配置，编辑后使用生成的`ASF.json`替换原来的文件即可

#### Using IPC GUI
ASF提供了一个IPC的GUI访问方式，默认这个功能是开启的，但是常用的功能都是支持的。

使用这个功能需要知道自己的`SteamOwnerID`，这个id可在[steamrep](https://steamrep.com/)网站查询，是一个7656开始的数字

也可以直接看自己的个人资料页面 `https://steamcommunity.com/profiles/`后面的数字就是

配置页面切换到ASF配置，配置全局配置文件`ASF.json`

```json
{
  "SteamOwnerID": "76561198099917059", 
  "UpdatePeriod": 0
}
```

1. 填入自己的`SteamOwnerID`
2. 在Remote Access中勾选IPC选项即可
3. 用新生成的`ASF.json`替换config目录的原始文件
4. 运行asf时，注意ipc服务是否有运行起来
![asf_ipc_run](/uploads/steam/asf_ipc_server.png)
5. 浏览器打开`http://127.0.0.1:1242/`就可以访问到asf的ipc界面


#### Command

##### 使用IPC执行命令
点击左侧的`Commands`, 在命令窗口输入命令，例如让所有的bot都添加游戏 输入

`!addlicense ASF 533150,533382,533349`

如果让指定的bot执行一个命令，需要在命令后指定bot的名称，

 `!addlicense  bottle_bot 884660`

* addlicensem命令后的id默认为subid，可以在steamdb上查到，如果要用app id，命令格式为

`addlicense ASF app/292030,sub/47807`

![asf_bot_command](/uploads/steam/asf_bot_command.png)

##### 使用与小号聊天执行命令

* 在生成bot的配置文件时，Access里面的SteamUserPermissions可以控制权限，权限有4种，默认为None。一般需要将自己帐号设置为Master最大权限。
* 每个命令有自己的权限要求，例如添加免费游戏的命令只需要operator权限

SteamUserPermissions是Key-Value格式的配置，key为用户的64位id，value为具体的权限数值，生成的配置文件部分如下：
```
  "SteamUserPermissions": {
    "76561198833106606": 3
  }
```

举例：
假设有大号Android和小号Apple，ASF中运行了一个大号Android的机器人bottle_bot。
在Steam网页上，大号Android发起与Apple的聊天，发起消息!addlicense bottle_bot 32287，则自动会把这个游戏加入到大号的库中。如果小号发送这个消息则没有任何反映，因为小号没有任何权限。这里小号的作用只是让大号可以把消息发给机器人而建立的聊天入口。因为大号无法自己给你聊天，除非通过群组聊天。

如果小号Apple也启动了一个`apple_bot`，则需要把Apple的64位id设置到`apple_bot`的用户权限中。在聊天窗口中执行`!addlicense 32287`，则所有的bot都会执行这个命令，根据发命令的用户的权限来判断是否执行这个命令。

在ASF全局配置中的设置的`SteamOwnerID`的帐号的权限为Owner拥有对于ASF中所有bot的最高权限，因此这个帐号可以让所有的bot执行所有的命令。一般这个id是大号的id，因此大号在聊天窗口中可以给所有的bot添加游戏执行命令。如果需要给指定bot发命令，则需要指明bot的名字。例如`!cmd bot_name param`

#### Privacy Policy
默认系统会使用你的帐号加入ASF群组



#### Plugins

##### ASFEnhance

[GitHub - chr233/ASFEnhance: ASF增强插件 / Add useful features for ASF](https://github.com/chr233/ASFEnhance)

将`ASFEnhance.dll` 丢进 ASF 目录下的 `plugins` 文件夹即可安装

2022 夏促，在网页的命令中输入

`EVENT ASF`,获取特卖徽章

`EVENTTHEME ASF`获取特卖主题

`EXPLORER ASF`5 秒后触发 ASF 探索队列任务