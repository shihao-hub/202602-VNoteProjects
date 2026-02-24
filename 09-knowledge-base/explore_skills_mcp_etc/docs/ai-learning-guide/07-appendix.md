# 附录

> 术语表、推荐资源、实践项目

---

## 术语表

### A

| 术语 | 英文 | 解释 |
|------|------|------|
| Agent | Agent | 智能体，能自主感知、决策、行动的 AI 系统 |
| Attention | Attention | 注意力机制，Transformer 的核心，让模型关注输入的不同部分 |

### B

| 术语 | 英文 | 解释 |
|------|------|------|
| Batch | Batch | 批处理，一次处理多个输入以提高效率 |

### C

| 术语 | 英文 | 解释 |
|------|------|------|
| Chain of Thought | Chain of Thought (CoT) | 思维链，让 LLM 展示推理过程的技巧 |
| Chunking | Chunking | 分块，将长文档切分成小段用于处理 |
| Context Window | Context Window | 上下文窗口，LLM 一次能处理的最大 Token 数 |
| Cosine Similarity | Cosine Similarity | 余弦相似度，衡量两个向量相似程度的方法 |

### E

| 术语 | 英文 | 解释 |
|------|------|------|
| Embedding | Embedding | 向量嵌入，将文本转换为数字向量的技术 |

### F

| 术语 | 英文 | 解释 |
|------|------|------|
| Few-shot Learning | Few-shot Learning | 少样本学习，通过几个示例让模型学习任务 |
| Fine-tuning | Fine-tuning | 微调，在预训练模型上用特定数据进一步训练 |

### G

| 术语 | 英文 | 解释 |
|------|------|------|
| Grounding | Grounding | 接地/落地，让 AI 输出基于真实数据而非纯生成 |

### H

| 术语 | 英文 | 解释 |
|------|------|------|
| Hallucination | Hallucination | 幻觉，LLM 生成看似合理但实际错误的内容 |
| Host | Host (MCP) | MCP 架构中的宿主，即运行 AI 的应用程序 |

### I

| 术语 | 英文 | 解释 |
|------|------|------|
| Inference | Inference | 推理，使用训练好的模型进行预测的过程 |

### L

| 术语 | 英文 | 解释 |
|------|------|------|
| LLM | Large Language Model | 大语言模型，基于大量文本训练的语言模型 |
| LoRA | Low-Rank Adaptation | 一种高效的模型微调方法 |

### M

| 术语 | 英文 | 解释 |
|------|------|------|
| MCP | Model Context Protocol | 模型上下文协议，AI 与外部服务的连接标准 |
| Memory | Memory | 记忆，Agent 存储和使用过去信息的能力 |
| Multimodal | Multimodal | 多模态，能处理文本、图像、音频等多种类型数据 |

### P

| 术语 | 英文 | 解释 |
|------|------|------|
| Parameter | Parameter | 参数，模型中可学习的数值，参数越多通常能力越强 |
| Prompt | Prompt | 提示词，给 AI 的输入指令 |
| Prompt Engineering | Prompt Engineering | 提示词工程，设计有效 Prompt 的技术和方法 |

### R

| 术语 | 英文 | 解释 |
|------|------|------|
| RAG | Retrieval-Augmented Generation | 检索增强生成，让 AI 先检索再回答的技术 |
| ReAct | Reasoning + Acting | 推理与行动结合的 Agent 模式 |
| Re-ranking | Re-ranking | 重排序，对初步检索结果进行精细排序 |

### S

| 术语 | 英文 | 解释 |
|------|------|------|
| Sampling | Sampling | 采样，LLM 生成文本时选择下一个 Token 的过程 |
| Server | Server (MCP) | MCP 架构中提供功能的服务端 |
| Skill | Skill | 技能，Agent 的专业能力模块 |
| System Prompt | System Prompt | 系统提示词，设定 AI 角色和行为的指令 |

### T

| 术语 | 英文 | 解释 |
|------|------|------|
| Temperature | Temperature | 温度，控制 LLM 输出随机性的参数 |
| Token | Token | 词元，LLM 处理文本的基本单位 |
| Tokenization | Tokenization | 分词，将文本转换为 Token 的过程 |
| Tool | Tool | 工具，Agent 可调用的外部功能 |
| Transformer | Transformer | 变换器，现代 LLM 的核心架构 |

### V

| 术语 | 英文 | 解释 |
|------|------|------|
| Vector Database | Vector Database | 向量数据库，专门存储和检索向量的数据库 |
| Vector Index | Vector Index | 向量索引，加速向量搜索的数据结构 |

### Z

| 术语 | 英文 | 解释 |
|------|------|------|
| Zero-shot Learning | Zero-shot Learning | 零样本学习，不提供示例直接让模型完成任务 |

---

## 推荐学习资源

### 官方文档

```
LLM 提供商文档
───────────────────────────────────────────
OpenAI
• 文档：https://platform.openai.com/docs
• 指南：https://platform.openai.com/docs/guides

Anthropic (Claude)
• 文档：https://docs.anthropic.com
• Prompt 指南：https://docs.anthropic.com/claude/docs

Google AI (Gemini)
• 文档：https://ai.google.dev/docs

MCP 协议
───────────────────────────────────────────
官方规范：https://spec.modelcontextprotocol.io
官方 Servers：https://github.com/modelcontextprotocol/servers
SDK 文档：https://modelcontextprotocol.io/docs

向量数据库文档
───────────────────────────────────────────
Pinecone：https://docs.pinecone.io
Milvus：https://milvus.io/docs
Chroma：https://docs.trychroma.com
Qdrant：https://qdrant.tech/documentation
```

### 优质博客与教程

```
入门级
───────────────────────────────────────────
• Hugging Face 课程
  https://huggingface.co/learn
  免费的 NLP 和 LLM 课程

• LangChain 文档和教程
  https://python.langchain.com/docs
  RAG 和 Agent 的实践框架

• LlamaIndex 文档
  https://docs.llamaindex.ai
  专注于 RAG 的框架

进阶级
───────────────────────────────────────────
• Lilian Weng 的博客
  https://lilianweng.github.io
  深入的 AI 技术解析

• Jay Alammar 的可视化解释
  https://jalammar.github.io
  图解 Transformer、GPT 等

• Chip Huyen 的 ML 博客
  https://huyenchip.com/blog
  机器学习工程实践

中文资源
───────────────────────────────────────────
• 动手学深度学习
  https://zh.d2l.ai
  李沐团队的深度学习教材

• 面向开发者的 LLM 入门课程
  https://github.com/datawhalechina/llm-universe
  Datawhale 出品
```

### 视频课程推荐

```
免费课程
───────────────────────────────────────────
• Andrew Ng - ChatGPT Prompt Engineering
  https://www.deeplearning.ai/short-courses/chatgpt-prompt-engineering-for-developers/
  Prompt 工程入门

• DeepLearning.AI 短期课程系列
  https://www.deeplearning.ai/short-courses/
  涵盖 LLM、RAG、Agent 等多个主题

• Stanford CS324 - Large Language Models
  https://stanford-cs324.github.io/winter2022/
  斯坦福 LLM 课程

B 站资源
───────────────────────────────────────────
• 李沐读论文系列
• 吴恩达课程中文翻译
• 各类 LLM 实战教程
```

### 实践平台

```
在线实验平台
───────────────────────────────────────────
• OpenAI Playground
  https://platform.openai.com/playground
  在线测试 GPT 模型

• Google AI Studio
  https://aistudio.google.com
  在线测试 Gemini 模型

• Hugging Face Spaces
  https://huggingface.co/spaces
  在线运行和分享 AI 应用

本地开发环境
───────────────────────────────────────────
• Ollama
  https://ollama.ai
  本地运行开源 LLM

• LM Studio
  https://lmstudio.ai
  图形化的本地 LLM 运行工具

• Text Generation WebUI
  https://github.com/oobabooga/text-generation-webui
  功能强大的本地 LLM 界面
```

---

## 实践项目建议

### 入门级项目

```
项目 1：Prompt 优化练习
───────────────────────────────────────────
目标：掌握基本的 Prompt 设计技巧

任务：
1. 选择一个任务（如代码解释、文章总结）
2. 编写初始 Prompt
3. 测试并记录结果
4. 应用学到的技巧优化 Prompt
5. 对比优化前后的效果

学习重点：
• 清晰性、上下文、格式指定
• Few-shot Learning
• 思维链技巧

预计时间：2-4 小时

───────────────────────────────────────────

项目 2：AI 对话应用
───────────────────────────────────────────
目标：了解 LLM API 的基本使用

任务：
1. 获取 OpenAI 或其他 LLM 的 API Key
2. 用 Python 编写简单的对话程序
3. 实现多轮对话（保持上下文）
4. 添加 System Prompt 定制 AI 性格

技术栈：
• Python
• OpenAI SDK 或其他 LLM SDK

示例代码结构：
```python
from openai import OpenAI

client = OpenAI()
messages = [
    {"role": "system", "content": "你是一个友好的助手"}
]

while True:
    user_input = input("你：")
    messages.append({"role": "user", "content": user_input})
    
    response = client.chat.completions.create(
        model="gpt-3.5-turbo",
        messages=messages
    )
    
    assistant_message = response.choices[0].message.content
    messages.append({"role": "assistant", "content": assistant_message})
    print(f"AI：{assistant_message}")
```

预计时间：4-6 小时
```

### 进阶级项目

```
项目 3：简单 Agent 搭建
───────────────────────────────────────────
目标：理解 Agent 的工作原理

任务：
1. 定义 2-3 个简单的工具（如搜索、计算）
2. 实现工具调用机制
3. 让 LLM 决定何时使用哪个工具
4. 实现多步骤任务执行

技术栈：
• Python
• LangChain 或自己实现

学习重点：
• 工具定义和描述
• Function Calling
• ReAct 模式

预计时间：6-10 小时

───────────────────────────────────────────

项目 4：个人知识库 RAG
───────────────────────────────────────────
目标：构建一个可以回答关于你自己文档问题的 AI

任务：
1. 准备一些文档（笔记、文章等）
2. 用 Embedding 模型向量化文档
3. 存入向量数据库（可以用 Chroma）
4. 实现检索 + 问答功能
5. 优化检索效果

技术栈：
• Python
• LangChain 或 LlamaIndex
• Chroma 或其他向量数据库
• OpenAI Embedding 或开源模型

关键步骤：
1. 文档加载和分块
2. 向量化和存储
3. 相似度检索
4. 构造 Prompt
5. LLM 生成回答

预计时间：8-12 小时
```

### 高级项目

```
项目 5：MCP Server 开发
───────────────────────────────────────────
目标：开发一个自定义的 MCP Server

任务：
1. 选择一个想让 AI 能访问的服务/数据
2. 使用 MCP SDK 创建 Server
3. 定义 Tools 和 Resources
4. 配置到 Claude Desktop 或其他 Host
5. 测试和迭代

示例场景：
• 日历管理 MCP Server
• 笔记应用 MCP Server
• 本地数据库 MCP Server

预计时间：10-20 小时

───────────────────────────────────────────

项目 6：多 Agent 协作系统
───────────────────────────────────────────
目标：构建多个 Agent 协作完成复杂任务

任务：
1. 设计 2-3 个专门化的 Agent
2. 实现 Agent 之间的通信
3. 设计协调机制
4. 测试复杂任务的完成情况

示例场景：
• 代码审查系统（代码分析 Agent + 安全审查 Agent + 报告生成 Agent）
• 内容创作系统（研究 Agent + 写作 Agent + 编辑 Agent）

技术栈：
• Python
• LangGraph 或 AutoGen
• 多种 LLM API

预计时间：20-40 小时

───────────────────────────────────────────

项目 7：生产级 RAG 系统
───────────────────────────────────────────
目标：构建一个可以部署使用的 RAG 应用

任务：
1. 设计文档处理 Pipeline
2. 实现高级检索（混合搜索、重排序）
3. 添加引用追溯功能
4. 实现流式输出
5. 添加用户反馈机制
6. 部署上线

技术栈：
• Python/FastAPI
• LangChain/LlamaIndex
• 生产级向量数据库
• 前端框架（可选）

预计时间：40-80 小时
```

---

## 学习路线总结

```
┌─────────────────────────────────────────────────────────────┐
│                    推荐学习路线                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  阶段 1：基础理解（1-2 周）                                 │
│  ────────────────────────                                   │
│  • 阅读模块一：理解 LLM 基本原理                           │
│  • 阅读模块二：掌握 Prompt Engineering                     │
│  • 练习：Prompt 优化项目                                   │
│                                                             │
│  阶段 2：Agent 入门（2-3 周）                               │
│  ────────────────────────                                   │
│  • 阅读模块三：理解 Agent 架构                             │
│  • 阅读模块四：了解 Skills 概念                            │
│  • 练习：简单 Agent 搭建项目                               │
│                                                             │
│  阶段 3：工具集成（2-3 周）                                 │
│  ────────────────────────                                   │
│  • 阅读模块五：掌握 MCP 协议                               │
│  • 练习：使用现有 MCP Server                               │
│  • 进阶：开发自定义 MCP Server                             │
│                                                             │
│  阶段 4：知识增强（2-4 周）                                 │
│  ────────────────────────                                   │
│  • 阅读模块六：理解 RAG 原理                               │
│  • 练习：个人知识库 RAG 项目                               │
│  • 进阶：生产级 RAG 系统                                   │
│                                                             │
│  阶段 5：综合应用（持续）                                   │
│  ────────────────────────                                   │
│  • 结合所学，构建实际应用                                  │
│  • 关注最新发展，持续学习                                  │
│  • 参与社区，分享经验                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 结语

恭喜你完成了这份学习指南！

现在你已经了解了：
- LLM 的基本原理和使用方法
- 如何通过 Prompt Engineering 有效与 AI 沟通
- Agent 和 Skills 如何让 AI 具备执行能力
- MCP 如何标准化 AI 与外部服务的连接
- RAG 如何让 AI 使用你的知识库

记住：
1. **理论与实践结合**：阅读之后一定要动手实践
2. **循序渐进**：从简单项目开始，逐步挑战复杂任务
3. **持续学习**：AI 领域发展迅速，保持关注和学习
4. **实际应用**：想想如何将所学应用到你的工作和生活中

祝你在 AI 学习之旅中一切顺利！

---

**返回 → [课程总览](./00-overview.md)**
