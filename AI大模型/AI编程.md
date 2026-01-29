一次聊天token组成：system prompt+用户问题+rules+对话历史+工具调用花费的token

IDE在的代码库检索Codebase Indexing，就是对整个代码仓库做RAG（将代码转换为可搜索的向量）



# Prompt

**System prompt：**描述当前AI的角色、性格、背景信息等定位（非用户想直接发生的消息）

**User prompt：**用户想直接询问的问题，发送给AI的话



# Function Call





# MCP





# Rules





# Skills

**skill的组成：**一个skill技能对应一个文件夹，一个技能通常包含skill.md文件、相应的文档、可运行脚本

**skill.md的组成：**skill.md文件为该技能的说明，由yaml头（技能名称、技能间接描述）、md格式的详细技能使用描述（在描述中又可以引用当前skill下其他文档、可运行脚本或者其他skill）组成。

**解决了token有限问题：**yaml头文件少，总是会被加载到上下文中；md格式的技能描述在AI认为需要使用该skill时触发加载，文件体积大小适中；AI使用该skill时按需使用描述中引用的当前skill下其他文档、可运行脚本或者其他skill。这样解决了上下文token受限的问题，避免了一个工具放在上下文中就占用大量token

## 与MCP的区别

skills就是简单的md文件；mcp创建需要编程、服务器配置

skills是渐进式加载按需加载工具；mcp在启用时就需要加载该mcp下所有工具

