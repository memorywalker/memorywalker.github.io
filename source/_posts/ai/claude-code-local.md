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
6. 运行`claude --model qwen3.5-9b-claude-4.6-opus-uncensored-distilled` 或 `claude --model gemma-4-e4b-it`   `claude --model qwen/qwen3.5-9b`

![claudecode](uploads/ai/claudecode.png)

7. 直接聊天让claude实现一个功能，这种方式纯聊天，只是在终端看文件的修改
![](uploads/ai/claudemakerustgame.png)

claude code现在加了一个宠物系统，输入`/buddy`命令时，命令会彩色显示，开启后，会显示显示一个宠物信息，并在会在终端输入框右侧放一个宠物图标，它会动态变化。我这里是一个稀有的蜗牛，名字叫Moth。宠物还有自己的属性，Deubg，Patience，Chaos，Wisdom，Snark 
![](uploads/ai/claudepet.png)

第三方API使用
```bash
export ANTHROPIC_API_KEY=sk-
export ANTHROPIC_AUTH_TOKEN=sk-
export ANTHROPIC_BASE_URL=https://
export ANTHROPIC_MODEL=claude-4.5-sonnet-2cc
export ANTHROPIC_DEFAULT_OPUS_MODEL=claude-4.5-sonnet-2cc
export ANTHROPIC_DEFAULT_SONNET_MODEL=claude-4.5-sonnet-2cc
export ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-4.5-sonnet-2cc
export CLAUDE_CODE_SUBAGENT_MODEL=claude-4.5-sonnet-2cc

如果要使用glm的模型，它兼容Claude Code
export ANTHROPIC_AUTH_TOKEN=sk-
export ANTHROPIC_BASE_URL=https://
export ANTHROPIC_DEFAULT_SONNET_MODEL=glm-4.7
export ANTHROPIC_DEFAULT_OPUS_MODEL=glm-4.7
export CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
export ENABLE_TOOL_SEARCH=0
export CLAUDE_CODE_DISABLE_EXPERIMENTAL_BETAS=1

anyrouter支持默认的claude模型，只需要配置这两个
export ANTHROPIC_BASE_URL=https://anyrouter.top
export ANTHROPIC_AUTH_TOKEN=sk-
```

### Tips

1. `/init` 命令可以让claude根据当前目录的文件自动推理出项目的作用，并生成一个说明文件`CLAUDE.md`，claude在这个目录中运行时上下文中都有这个文件内的信息。
2. `./.claude/skills`中是旨在当前项目中加载的skills，而`~/.claude/skills`则是全局可以使用的skills
3. . `/agents` 创建子agent，当一个会话agent做的事情太多，可以把它的任务分拆给多个子agent来工作，减少主agent的上下文的数据量，例如主agent用来开发实现，一个子agent用来代码评审，一个子agent用来执行单元测试。创建出来的子agent在项目目录的`./.claude/agents/xxx.md`，每一个子agent有一个自己的agent名字的md文件。注意子agent在被指定了开始执行任务后，它会加载它要使用的skills的完整的`SKILL.md`文件的内容，而不只是文件头信息。在这个md文件中可以指定子agent可以会使用的tools, model, skill。例如code-reviewer.md文件头如下：
    ```
    ---
    name: code-reviewer
    description: "Reviews code for quality, security, and convention compliance. Use when user asks to review, check, or verify code"
    tools: Bash, Glob, Grep, Read
    model: inherit
    color: purple
    skills: reviewing-cli-command
    ---
    ```
使用类似`use the code-reviewer subagent to review the code @../src/main.rs`来指派一个subagent同时工作

### 总结

1. 对于想体验在Claude使用本地模型或者第三方模型是可行的
不过本地模型太小，处理太慢，不确定是不是模型适配的问题，但是如果直接在lm studio中提问，立即就可以回答。
使用google的 `gemma-4-e4b-it`比qwen的要快一点，但是结果拉很多。还是得用在线服务商或者找个[公益站](https://elysiver.h-e.top)比较好。

2. 对于会编码的人使用cli来实现功能，效率太低了，有些错误在IDE中很容易就可以自己修改，使用AI反而要思考改来改去，当然也和我用的模型比较差有关。但是如果会编程，使用IDE的版本效率肯定还是高的。Vibe Coding还是适合一点都不会编程或没有IDE的场景。
3. 可以在LM Studio的开发者日志窗口中看到Claude与模型的交互，提示词量很大，如果是本地模型上下文需要配置大一些，如果使用在线以token为单位计费，成本应该很高，但是应该比人的工资低


