---
title: Agent Skills
date: 2026-04-06T09:24:00
categories:
  - AI
tags:
  - AI
---

## Agent Skills

官网 https://agentskills.io/home

吴恩达Agent Skills视频课程 https://www.bilibili.com/video/BV1bE6iB7EFG

### 概念

Agent Skills are a lightweight, open format for extending AI agent capabilities. A skill is a folder of organized files consisting of instructions, scripts, assets, and resources that agents can discover to perform specific tasks accurately.
Skill是Antrhopic公司提出的开放标准，现在很多agent都支持，所以开发出来的skill可以分享给不同的人和公司使用。

#### 发展诉求

* **过去** 根据需要开发不同的专家Agent
比如代码Agent，研究Agent，金融Agent，他们每一个都有自己的专注点，专用工具和脚手架(流程)
这些专家Agent本质上工作模式是相同的，基于特定的上下文和领域知识，调用专用的工具，那么**可以把这个工作模式提炼成一个标准模式**

* **现在** 一个简单通用目的的agent，但是有很多不同的技能
它使用bash和文件系统读写，它依赖上下文和领域专家来完成工作，它通过Skills提供的流程规范和专家上下文信息，这个通用Agent在需要的时候加载这些信息和工具。
#### 什么时候使用skill？

当Agent作为以下作用时，可以通过Skill来实现，甚至可以把多个skill组合起来，完成一个复杂的工作流。

**领域专家**：例如品牌指南和模板，法律审查，数据分析

**可重复的工作流**：**非确定系统**中，每次输入，模型返回的结果可能是不同的，导致结果不是我们预期的，通过skill中明确的步骤和说明，从而让agent可以输出可以预期的结果。例如每周项目总结，客服应答工作流，季报回顾等

**新的技能**：例如创建ppt，文件格式转换，构建MCP Server

当你有一个工作流，你需要依次的向Agent发送请求，这个工作流每次都一样的步骤，每次都需要描述的指令和需求，告诉agent参考资料和使用的工具，需要人工确认工作流和结果是一致的。

直观的判断，你每次都需要在不同的会话中输入相同的提示词，上下文和工具，这时就可以把这些提示词，上下文，工具打包成一个Skill。

### Agent/Skill/MCP/Subagent/Tools/LLM关系

他们相互协作共同完成任务，不存在谁好谁坏。

**Prompt**：与LLM进行对话，输入信息，获取模型的反馈
**MCP**： 连接agent与外部系统和数据，例如查询数据库，API获取实时数据
**Skill**：告诉agent怎么使用拿到的数据，通过专家知识扩展Agent的能力，在需要的时候使用工具
**工具**：给agent提供完成任务的能力，工具的名称，描述和参数加载在上下文中
**子agent**：每一子agent有自己的上下文和工具权限，子Agent可以并行工作，例如一个专业的代码评审agent


![](uploads/ai/agentskillmcp.png)

举例：一个客户反馈分析

Skill 提供如何分类反馈，并总结结论
MCP 通过google文档API获取用户调查表数据信息
两个子Agent分别处理用户的面谈和表格调查分析

| 特性   | Skills                      | Prompts | Subagents  | MCP  |
| ---- | --------------------------- | ------- | ---------- | ---- |
| 作用   | 流程知识                        | 时时刻刻的指令 | 任务委派       | 工具资源 |
| 持续性  | 跨对话                         | 单个对话    | 跨会话        | 持续连接 |
| 组成   | Instructions, code, assests | 自然语言    | 完整的agent逻辑 | 工具定义 |
| 加载   | 动态按需加载                      | 每一轮都加载  | 被调用时加载     | 随时可用 |
| 适用场景 | 专家系统                        | 快速请求    | 具体专门的任务    | 数据访问 |


### 如何工作

一个Skill会有大量信息，而我们可能会用很多个skill，为了减少上下文的数据量Skills are **progressively disclosed** **渐进式披露**to the agent. 

它的名称和描述始终在上下文，但是其他的内容只有在用户请求和skill的描述匹配时候才会被加载到上下文，从而避免提示词太多。

从它的功能上可以推出要使用Skill，Agent需要具备**读写文件**，以及**批处理工具来执行代码**的能力。

### 组织结构 

一个skill通常有三个部分组成：
1.  **Metadata** (YAML格式: name, description)，必须， 始终加载到上下文中
2.  **Instructions** (SKILL.md 正文内容)，必须， 模型检测到匹配时加载 
3.  **Resources** (参考文件，脚本等) ，可选，需要时加载

由于pdf这个skill在Agent Skills成为开放标准之前就写好了，所以它的组织并没有严格按标准。
```
analyzing-marketing-campaign/
├── SKILL.md
└── references/
    └── budget_reallocation_rules.md

pdf/
├── SKILL.md
├── forms.md
├── reference.md
└── scripts/
    ├── check_fillable_fields.py
    ├── convert_pdf_to_images.py
    ├── extract_form_field_info.py
    └── fill_pdf_form_with_annotations.py

designing-newsletters/
├── SKILL.md
├── references/
│   └── style-guide.md
└── assets/
    ├── header.png
    ├── icons/
    └── templates/
        ├── newsletter.html
        └── layout.docx
```
#### skill.md 

文件头都有一个**yaml格式**的说明这个skill的**名称**和**描述**。
```yaml
---
name: analyzing-market 这个skill的名称，规则：单词全部小写，单词之间使用-连接，不能使用关键字，例如claude或anthropic
description: Analyze weekly marketing campaign performance data across channels. LLM通过分析这个描述来决定什么时候使用这个skill
---
```
详细的流程**说明指南**在正文中，包括需求，输入，输出，以及当满足某些条件后，可以引用其他文件，例如可执行脚本，其他markdown文件，模板，图片等资源文件。


### 最佳实践

https://agentskills.io/skill-creation/best-practices
https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf?hsLang=en

#### SKILL.md

这个文件有两部分组成：
1. **YAML Frontmatter** 顶部元信息
2. **Body Content** 正文说明

##### name

* Max 64 chars; 
* lowercase letters, numbers, and hyphens only;短连接线`-`
* must not start/end with hyphens; 
* must match parent directory name; 
* recommended: gerund (verb+-ing) form 动词的ing形式

##### description

* Max 1024 chars; 
* non-empty; 
* should describe **what the skill does AND when to use** it; 
* include **specific keywords to help agents identify** relevant tasks

YAML中其他可选字段

| Field | Constraints |
|-------|-------------|
| **license** | License name or reference to a license file |
| **compatibility** | Max 500 chars; indicates environment requirements |
| **metadata** | Arbitrary key-value pairs (e.g., author, version) |
| **allowed-tools** | Space-delimited list of pre-approved tools (Experimental) |

##### 正文说明

markdown格式内容，建议包括：
- 工作流一步一步的说明，如果一个步骤是可以跳过的，也要明确写出来
- 输入格式，输出格式以及举个例子
- 常规的边界情况

**实践经验**

- 内容小于500行
- 把参考资料放到独立的文件中，只展示基本内容，连接到更深入的资料
- 参考资料只保持一层引用，不嵌套多层引用
- 保持清晰和明确，使用一致的术语
- 文件路径中使用`/`，linux系统的路径风格
- 复杂的工作流分解为多个清晰有序的独立步骤的skill，要比一个特别大的skill更有价值


**自由度**

一个skill可以发挥多大的自由度，例如设计PPT的颜色，字体可以有多个不同的选择。

| Level              | Description                                 |
| ------------------ | ------------------------------------------- |
| **High freedom**   | 通用基于文本的指令; 多种方法都是有效的                        |
| **Medium freedom** | 说明包含自定义的伪代码，代码例子或模式；提供一个偏好的模式，但是它的变体也是可以接受的 |
| **Low freedom**    | 说明引用的具体的脚本；必须遵循的顺序流程                        |

### 可选的目录

#### `/assets`
- **Templates:** 文档模板, 配置模板
- **Images:** 图, logos
- **Data files:** 数据表, 模式信息

#### `/references`
- agent需要读取的额外文档资料
- 一个文件只描述一件事情
- 超过100行的文件，在文件开始添加一个目录TOC，这样agent可以知道全局
#### `/scripts`
- 清晰的文档依赖
- 脚本有清晰的文档说明
- 错误处理要明显和有用的
- 说明中要明确告诉agent是执行这个脚本还是把它作为一个参考资料
### 评估

- 人工评估反馈好坏
- 使用所有会用到的模型进行评估

#### 单元测试

单元测试和软件的测试类似，一个测试用例包括:
- **skills**: 要测试哪个skill
- **queries**: 执行测试的prompt
- **files**: 输入的文件
- **expected_behavior**: 预期的结果

#### Example Test Case
```json
{
  "skills": ["generating-practice-questions"],
  "queries": [
    "Generate practice questions from this lecture note and save it to output.md",
    "Generate practice questions from this lecture note and save it to output.tex",
    "Generate practice questions from this lecture note and save it to output.pdf"
  ],
  "files": ["test-files/notes.pdf", "test-files/notes.tex", "test-files/notes.pdf"],
  "expected_behavior": [
    "Successfully reads and extracts the input file. For pdf input, uses pdfplumber.",
    "Successfully extracts all the learning objectives.",
    "Generates the 4 types of questions.",
    "Follows the guidelines for each question.",
    "Uses the output structure and the correct output templates.",
    "The latex output successfully compiles.",
    "Saves the generated questions to a file named output."
  ]
}
```


