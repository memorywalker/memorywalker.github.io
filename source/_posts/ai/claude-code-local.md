---
title: Claude Code使用本地模型
date: 2026-04-04T21:37:00
categories:
  - AI
tags:
  - AI
---
## Claude Code使用

### Cluade Code 安装

1. 安装node.js 一般开发机器都会安装
2. 安装Claude Code `npm install -g @anthropic-ai/claude-code`，使用`claude --version`查看版本
3. 运行LM Studio，并开启服务，务必更新LM Studio的版本到0.4.9（目前最新版本），不然claude响应很慢，还会卡住
4. 系统环境变量增加git-bash的路径 `CLAUDE_CODE_GIT_BASH_PATH=D:\Program Files\Git\bin\bash.exe`
5. 设置claude cli的环境变量
    ```bash
    export ANTHROPIC_BASE_URL=http://localhost:1234
    export ANTHROPIC_AUTH_TOKEN=lmstudio
    ```
6. 运行`claude --model qwen3.5-9b-claude-4.6-opus-uncensored-distilled` 或 `claude --model gemma-4-e4b-it`

![claudecode](uploads/ai/claudecode.png)

7. 直接聊天让claude实现一个功能，这种方式纯聊天，只是在终端看文件的修改
![](uploads/ai/claudemakerustgame.png)

不过本地模型太小，处理太慢，不确定是不是模型适配的问题，但是如果直接在lm studio中提问，立即就可以回答。
使用google的 `gemma-4-e4b-it`比qwen的要快一点，但是结果拉很多。

还是得用在线服务商的才行。公益站 https://elysiver.h-e.top

