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

claude code现在加了一个宠物系统，输入`/buddy`命令时，命令会彩色显示，开启后，会显示显示一个宠物信息，并在会在终端输入框右侧放一个宠物图标，它会动态变化。我这里是一个稀有的蜗牛，名字叫Moth。宠物还有自己的属性，Deubg，Patience，Chaos，Wisdom，Snark 
![](uploads/ai/claudepet.png)

### 总结

1. 对于想体验在Claude使用本地模型或者第三方模型是可行的
不过本地模型太小，处理太慢，不确定是不是模型适配的问题，但是如果直接在lm studio中提问，立即就可以回答。
使用google的 `gemma-4-e4b-it`比qwen的要快一点，但是结果拉很多。还是得用在线服务商或者找个[公益站](https://elysiver.h-e.top)比较好。

2. 对于会编码的人使用cli来实现功能，效率太低了，有些错误在IDE中很容易就可以自己修改，使用AI反而要思考改来改去，当然也和我用的模型比较差有关。但是如果会编程，使用IDE的版本效率肯定还是高的。Vibe Coding还是适合一点都不会编程或没有IDE的场景。
3. 可以在LM Studio的开发者日志窗口中看到Claude与模型的交互，提示词量很大，如果是本地模型上下文需要配置大一些，如果使用在线以token为单位计费，成本应该很高，但是应该比人的工资低


