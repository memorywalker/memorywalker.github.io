---
title: VS Code 工具
date: 2025-07-30 23:25:49
categories:
- program
tags:
- vs ocde
- LSP
- DAP
---


## VS Code工具

### Language Server Protocol 

https://microsoft.github.io/language-server-protocol/overviews/lsp/overview/

代码编辑器中常用的自动补全，转到定义，浮动相关显示文档的功能，每个编辑工具对每种语言都有一套自己的实现，这是很大的重复工作。

通过对每种语言提供一个这个语言规范话的服务端，编辑工具通过与这个服务端进程间通信实现常见的功能。*Language Server Protocol (LSP)* 定义服务与开发工具的通信协议规范，这样服务端可以被多个不同的编辑工具服用。

#### 工作流程

语言服务器作为一个独立的进程运行，开发工具根据语言协议通过JSON-RPC与语言服务器通信。

下面是开发工具和语言服务之间简单的交互过程，包括了打开文档，编辑文档，转到定义以及关闭文档。交互中使用的数据只是文本文档的URI和文档中的位置信息，这些数据是编程语言无关的，所以更容易标准化。

   ![language-server-sequence](../../uploads/tech/language-server-sequence.png)
   ![language-server-sequence](/uploads/tech/language-server-sequence.png)   

* 当开发工具通知了语言服务打开文档后，这份文档的内容在开发工具的管理的内存中维护，同时确保它和语言服务是同步更新的。
* 用户编辑了文档后，开发工具通知语言服务文档变化信息，语言服务通过分析变化的代码返回诊断信息，例如编译警告或错误
* 转到定义点击后，开发工具给语言服务发送转到定义请求，并附带当前文档的URI和‘Go to Definition’ 所点击的文本位置给语言服务，语言服务再把函数定义的文档URI和函数定义的文本位置返回给开发工具
* 当关闭文件后，开发工具通知语言服务这个文档已经不在内存中了，磁盘文件系统中的文件就是最新的文件。

这个开发工具转到定义的请求

```json
{
    "jsonrpc": "2.0",
    "id" : 1,
    "method": "textDocument/definition",
    "params": {
        "textDocument": {
            "uri": "file:///p%3A/mseng/VSCode/Playgrounds/cpp/use.cpp"
        },
        "position": {
            "line": 3,
            "character": 12
        }
    }
}
```

语言服务应答信息为

```json
{
    "jsonrpc": "2.0",
    "id": 1,
    "result": {
        "uri": "file:///p%3A/mseng/VSCode/Playgrounds/cpp/provide.cpp",
        "range": {
            "start": {
                "line": 0,
                "character": 4
            },
            "end": {
                "line": 0,
                "character": 11
            }
        }
    }
}
```

#### 语言服务

当开发工具中打开了多种编程语言的文件，开发工具会给每一种语言启动一个语言服务，所以VS Code中打开的语言类型越多，越耗费资源。

语言服务如何集成在开发工具中由开发工具来决定。微软提供了如何实现一个语言Server的指南

https://code.visualstudio.com/api/language-extensions/language-server-extension-guide

   ![language-server](../../uploads/tech/language-server.png)
   ![language-server](/uploads/tech/language-server.png)   

### Debug Adapter Protocol

https://microsoft.github.io/debug-adapter-protocol/overview

客户端和Debug Adapter交互流程   ![init-launch](../../uploads/tech/init-launch.svg)
   ![init-launch](/uploads/tech/init-launch.svg)   