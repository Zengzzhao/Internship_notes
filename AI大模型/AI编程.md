一次聊天token组成：system prompt+用户问题+rules+对话历史+工具调用花费的token

IDE在的代码库检索Codebase Indexing，就是对整个代码仓库做RAG（将代码转换为可搜索的向量）



# claude code

```bash
# 查看可用的子agent
/agents

# 终端对话引用图像
# 1.拖拽图片到cli；2.复制图片后粘贴到cli；3.在cli中提供图片路径

# 引用文件、目录、子agents
# 通过@符引用

```

## 子agent

### 内置子agent

Explore：使用便宜的Haiku模型快速、只读地搜索分析代码库

Plan：继承主对话的模型，只读模式对代码进行规划

General-purpose：继承主对话的模型，可读可写处理任务

Bah：继承主对话的模型，当在单独的上下文中运行终端命令时调用

statusline-setup：使用Sonnet模型，当运行 `/statusline` 来配置状态行时调用

Claude Code Guide：使用Haiku模型，当提出关于Claude Code功能的问题时调用

### 创建自定义子agent

通过/agents命名创建，或者书写带有YAML前置元数据的MD文件

全局个人agents：所有项目都可以使用的agents，存放在`~/.claude/agents/`下

项目agents：仅当前项目可以使用的agents，存放在每个项目的`.claude/agents`下





# Prompt

**System prompt：**描述当前AI的角色、性格、背景信息等定位（非用户想直接发生的消息）

**User prompt：**用户想直接询问的问题，发送给AI的话



# Function Call





# MCP



# RAG

## 索引Index

步骤：

将知识内容分割Split

嵌入Embedding：使用嵌入模型将文本内容转换为一个嵌入向量，核心逻辑就是将文本token化，对每个token计算其嵌入向量，将文本对应的所有嵌入向量池化得到一个平均嵌入向量

存储Storage：存储嵌入结果到向量数据库

## 检索Retrieval

将用户的问题与建立的索引进行匹配，得到相关度高的结果

## 生成Generation

将匹配到的知识与用户问题混合给LLM，得到最终结果



# Rules





# Skills

**skill的组成：**一个skill技能对应一个文件夹，一个技能通常包含skill.md文件、相应的文档、可运行脚本

**skill.md的组成：**skill.md文件为该技能的说明，由yaml头（技能名称、技能间接描述）、md格式的详细技能使用描述（在描述中又可以引用当前skill下其他文档、可运行脚本或者其他skill）组成。

**解决了token有限问题：**yaml头文件少，总是会被加载到上下文中；md格式的技能描述在AI认为需要使用该skill时触发加载，文件体积大小适中；AI使用该skill时按需使用描述中引用的当前skill下其他文档、可运行脚本或者其他skill。这样解决了上下文token受限的问题，避免了一个工具放在上下文中就占用大量token

## 与MCP的区别

skills就是简单的md文件；mcp创建需要编程、服务器配置

skills是渐进式加载按需加载工具；mcp在启用时就需要加载该mcp下所有工具

