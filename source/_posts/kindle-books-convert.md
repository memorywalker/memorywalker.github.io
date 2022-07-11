---
title: kindle books convert
date: 2022-06-19 18:25:49
categories:
- read
tags:
- kindle
- book
---

### Kindle Books Convert

最近开通了美区Amazon Prime会员试用一个月，因此有了Prime Reading的福利，同时亚马逊中国也宣布了2024年6月kindle退出中国，以后如果需要同步电子书，只能使用美区的帐号同步美区的服务。

Prime Reading一个用户可以一次租借10本书，主要是小说和杂志

### 转换官方电子书 (Epubor Ultimate)

1. 下载`EpuborUltimate` 
2. 下载`KindleForPC-installer-1.17.44170.exe`，注意`EpuborUltimate` 会检测Kindle For PC的版本，如果是1.25之后的版本，会提示进行版本降级，并在[帮助文档]([Welcome to Epubor Knowledge Base (FAQ)](https://www.epubor.com/faq.html?utm_medium=soft&utm_source=right_menu&utm_campaign=kindle&utm_content=EpuborUltimatev3.0.12.529_36498-15#e501))中提供[下载地址](http://download.epubor.com/KindleForPC-installer-1.17.44170.exe?_ga=2.245238471.374470622.1655634956-1642488113.1655634956)
3. 在PC版本的Kindle软件登录后，可以看到自己库中的所有电子书，把需要转换的电子书下载下来
4. 在`EpuborUltimate`中配置好Kindle电子书的目录后，在软件中可以看到当前的书，选择一本书，拖入右侧的工作区后，在下方选择需要转换的格式，就可以进行转换了
5. 如果出现转换失败，可以升级最新版本的`EpuborUltimate`软件，在设置中可以自动下载升级包。



我的百度网盘

Software/kindle/Kindle 正版书转换工具/

`EpuborUltimate`

`KindleForPC-installer-1.17.44170.exe`



### 转换官方电子书 (Calibre)

不知道为什么昨天还能使用`EpuborUltimate`转换电子书，今天就提示软件不支持租借来的电子书，那就换开源的Calibre

1. 下载Calibre的3.48版本，之后的版本不支持win7运行 [calibre release (3.48.0) (calibre-ebook.com)](https://download.calibre-ebook.com/3.48.0/)
2. 下载插件[DeDRM_tools](https://github.com/apprenticeharper/DeDRM_tools)，对于calibre 4.x and earlier，需要下载[v6.8.1](https://github.com/apprenticeharper/DeDRM_tools/releases/tag/v6.8.1)；下载好后，把压缩包解压，其中有`DeDRM_Plugin.zip`这个文件
3. 运行Calibre，到Preference中，高级，插件，选择Load Plugin from files，选择刚刚的`DeDRM_Plugin.zip`，安装后重启Calibre程序
4. 把Kindle库目录的azw格式的电子书拖入Calibre后，解析完成后就已经去掉了DRM，可以右键选择这本书，转换格式为mobi



