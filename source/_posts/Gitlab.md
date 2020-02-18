---
title: Gitlab使用
date: 2020-02-18 20:25:49
tags: Gitlab; git
---


## Gitlab

 https://gitlab.com/ 

Gitlab实现了git flow的工作模式，可以进行项目的管理、追溯、任务分配。

可以在网站注册账号直接使用gitlab的服务，也可以下载软件，自己在linux系统安装配置服务

注册时需要人机验证，需要科学上网

### 远程仓库

使用账号登陆后，可以开始创建一个项目

这个项目可以自己从零开始创建，也可以使用现有的模板，甚至从其他平台如GitHub导入

项目创建完成后，就可以`git clone`下来再本地进行开发了

### 项目管理

### Milestone

可以看做是一个大的功能版本，这个版本里面有一些小的功能Issue组成

例如可以把读一本书作为一个里程碑

新建一个里程碑时，可以设置**标题**和**开始**、**结束日期**

### Issue

一个Issue是一个独立的功能点，例如可以是读完书的某一个章节

* 一个Issue可以把它指派给某个成员，这个成员的To Do List将会收到通知

* 可以把它设置为某个milestone的issue
* issue可以设置完成时间

直接在To Do List里点击对应的Issue，就可以看Issue的信息

#### 处理Issue

本地新建一个对应Issue的分支`git checkout -b wireshark`

代码完成后，本地commit之后，push到远端

`git push --set-upstream origin wireshark` 

填写commit的消息时，可以填入issue的编号例如`read chapter 1 finished #1.`其中的`#1`可以自动关联到对应的issue

此时在第一个issue的信息页面可以看到

```markdown
Memory Walker @memorywalker changed due date to February 22, 2020 11 minutes ago
Memory Walker @memorywalker changed milestone to %wireshark数据包分析 11 minutes ago
Memory Walker @memorywalker mentioned in commit 57932869 5 minutes ago
```

在Merge Request中新建一个Request，选择issue的分支合并到master，并选择对应的管理人进行合并

管理人会收到一个新的Merge Request的任务，可以自己或再找人审核提交的内容

在changes标签页可以看到更改的内容，并进行评注

如果没有问题，可以点击merge进行合并，然后就可以关闭这个issue

#### 测试项目

 https://gitlab.com/memorywalker/blog/ 











