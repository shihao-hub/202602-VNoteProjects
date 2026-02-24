# 进阶模块二：Agent 开发实战

> 使用主流框架构建 AI Agent 应用

---

## 第28章：Agent 开发框架概览

### 28.1 为什么需要框架

```
自己从头实现 Agent 的挑战：

1. 工具调用管理
   • 工具定义格式
   • 参数解析和验证
   • 错误处理

2. 对话管理
   • 上下文维护
   • 多轮对话状态
   • 记忆管理

3. 流程控制
   • 复杂流程编排
   • 条件分支
   • 循环和重试

4. 集成
   • 各种 LLM API
   • 向量数据库
   • 外部工具和服务

框架帮你解决这些问题，让你专注于业务逻辑
```

### 28.2 主流框架对比

```
┌─────────────────────────────────────────────────────────┐
│                   Agent 开发框架对比                    │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  LangChain                                              │
│  ─────────                                              │
│  特点：                                                 │
│  • 最流行的 LLM 应用框架                               │
│  • 功能全面，生态丰富                                  │
│  • 抽象层次高，上手快                                  │
│  • 社区活跃，文档丰富                                  │
│                                                         │
│  适合：                                                 │
│  • 快速原型开发                                        │
│  • RAG 应用                                            │
│  • 需要丰富集成的项目                                  │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  LlamaIndex                                             │
│  ──────────                                             │
│  特点：                                                 │
│  • 专注于数据索引和检索                                │
│  • RAG 能力最强                                        │
│  • 支持多种数据源                                      │
│  • 与 LangChain 可配合使用                             │
│                                                         │
│  适合：                                                 │
│  • 知识库问答                                          │
│  • 文档处理                                            │
│  • 复杂数据检索                                        │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  OpenAI Assistants API                                  │
│  ─────────────────────                                  │
│  特点：                                                 │
│  • OpenAI 官方方案                                     │
│  • 托管服务，无需管理基础设施                          │
│  • 内置代码解释器、文件检索                            │
│  • 简单易用                                            │
│                                                         │
│  适合：                                                 │
│  • 快速上线                                            │
│  • 简单 Agent 需求                                     │
│  • 不想管理复杂性                                      │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  Semantic Kernel                                        │
│  ───────────────                                        │
│  特点：                                                 │
│  • 微软出品                                            │
│  • 企业级特性                                          │
│  • 多语言支持（Python/C#/Java）                        │
│  • 与 Azure 集成好                                     │
│                                                         │
│  适合：                                                 │
│  • 企业应用                                            │
│  • .NET 生态                                           │
│  • Azure 用户                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 第29章：LangChain 实战

### 29.1 LangChain 核心概念

```
LangChain 的核心组件：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  Model I/O（模型交互）                                  │
│  ─────────────────────                                  │
│  • Prompts：提示模板管理                               │
│  • LLMs/Chat Models：各种模型的统一接口                │
│  • Output Parsers：输出解析                            │
│                                                         │
│  Retrieval（检索）                                      │
│  ─────────────────                                      │
│  • Document Loaders：加载各种格式文档                  │
│  • Text Splitters：文档分块                            │
│  • Embeddings：向量化                                  │
│  • Vector Stores：向量数据库集成                       │
│  • Retrievers：检索器                                  │
│                                                         │
│  Chains（链）                                           │
│  ───────────                                            │
│  • 将多个组件串联成流程                                │
│  • 预定义的常用链（如 RAG 链）                         │
│  • 自定义链                                            │
│                                                         │
│  Agents（智能体）                                       │
│  ────────────────                                       │
│  • 让 LLM 决定使用哪些工具                             │
│  • 多种 Agent 类型                                     │
│  • 工具定义和管理                                      │
│                                                         │
│  Memory（记忆）                                         │
│  ──────────────                                         │
│  • 对话历史管理                                        │
│  • 不同记忆策略                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 29.2 基础使用示例

```python
# 安装
# pip install langchain langchain-openai

# === 基本对话 ===
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage

# 创建模型
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.7)

# 简单对话
messages = [
    SystemMessage(content="你是一个友好的助手"),
    HumanMessage(content="你好，请介绍一下自己")
]
response = llm.invoke(messages)
print(response.content)


# === 使用 Prompt 模板 ===
from langchain_core.prompts import ChatPromptTemplate

# 定义模板
prompt = ChatPromptTemplate.from_messages([
    ("system", "你是一个{role}，请用{style}的方式回答问题"),
    ("human", "{question}")
])

# 使用模板
chain = prompt | llm  # LCEL 语法：管道符连接组件

response = chain.invoke({
    "role": "Python 专家",
    "style": "简洁专业",
    "question": "什么是列表推导式？"
})
print(response.content)


# === 输出解析 ===
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field

# 定义输出结构
class CodeReview(BaseModel):
    issues: list[str] = Field(description="发现的问题列表")
    suggestions: list[str] = Field(description="改进建议")
    score: int = Field(description="代码质量评分 1-10")

# 创建解析器
parser = JsonOutputParser(pydantic_object=CodeReview)

# 使用解析器
prompt = ChatPromptTemplate.from_messages([
    ("system", "审查以下代码并以 JSON 格式输出结果。\n{format_instructions}"),
    ("human", "{code}")
])

chain = prompt | llm | parser

result = chain.invoke({
    "code": "def add(a, b): return a+b",
    "format_instructions": parser.get_format_instructions()
})
print(result)  # 返回结构化的字典
```

### 29.3 构建 RAG 应用

```python
# 安装依赖
# pip install langchain langchain-openai chromadb

from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import Chroma
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# === 步骤 1：加载文档 ===
loader = TextLoader("knowledge.txt", encoding="utf-8")
documents = loader.load()

# === 步骤 2：文档分块 ===
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # 每块大小
    chunk_overlap=50,    # 重叠部分
    separators=["\n\n", "\n", "。", "，", " ", ""]
)
splits = text_splitter.split_documents(documents)
print(f"分成了 {len(splits)} 块")

# === 步骤 3：创建向量存储 ===
embeddings = OpenAIEmbeddings()
vectorstore = Chroma.from_documents(
    documents=splits,
    embedding=embeddings,
    persist_directory="./chroma_db"  # 持久化目录
)

# === 步骤 4：创建检索器 ===
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}  # 返回 top 3 相关文档
)

# === 步骤 5：构建 RAG 链 ===
llm = ChatOpenAI(model="gpt-4o-mini")

# RAG 提示模板
prompt = ChatPromptTemplate.from_template("""
基于以下参考资料回答问题。如果资料中没有相关信息，请说明。

参考资料：
{context}

问题：{question}

回答：
""")

# 辅助函数：格式化检索结果
def format_docs(docs):
    return "\n\n".join(doc.page_content for doc in docs)

# 构建 RAG 链
rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

# === 步骤 6：使用 ===
question = "请介绍一下项目的主要功能"
answer = rag_chain.invoke(question)
print(answer)
```

### 29.4 构建 Agent

```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_core.tools import tool
from langchain_community.tools import DuckDuckGoSearchRun

# === 步骤 1：定义工具 ===

@tool
def calculator(expression: str) -> str:
    """计算数学表达式。输入应该是一个有效的数学表达式。"""
    try:
        result = eval(expression)
        return f"计算结果：{result}"
    except Exception as e:
        return f"计算错误：{str(e)}"

@tool  
def get_current_time() -> str:
    """获取当前时间。"""
    from datetime import datetime
    return datetime.now().strftime("%Y-%m-%d %H:%M:%S")

# 使用社区工具
search = DuckDuckGoSearchRun()

tools = [calculator, get_current_time, search]

# === 步骤 2：创建 Agent ===
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0)

prompt = ChatPromptTemplate.from_messages([
    ("system", """你是一个有用的助手，可以使用工具来帮助用户。

可用工具：
- calculator：计算数学表达式
- get_current_time：获取当前时间
- duckduckgo_search：搜索互联网

根据用户的问题，决定是否需要使用工具，以及使用哪个工具。"""),
    MessagesPlaceholder(variable_name="chat_history", optional=True),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad")
])

# 创建 Agent
agent = create_tool_calling_agent(llm, tools, prompt)

# === 步骤 3：创建 Agent 执行器 ===
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,  # 打印执行过程
    handle_parsing_errors=True,  # 处理解析错误
    max_iterations=5  # 最大迭代次数
)

# === 步骤 4：使用 ===
# 简单计算
result = agent_executor.invoke({"input": "123 * 456 等于多少？"})
print(result["output"])

# 搜索信息
result = agent_executor.invoke({"input": "今天的科技新闻有哪些？"})
print(result["output"])

# 复合任务
result = agent_executor.invoke({
    "input": "现在几点了？另外帮我计算一下从现在到晚上 8 点还有多少分钟"
})
print(result["output"])
```

### 29.5 添加对话记忆

```python
from langchain_core.chat_history import BaseChatMessageHistory
from langchain_community.chat_message_histories import ChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# 存储会话历史
store = {}

def get_session_history(session_id: str) -> BaseChatMessageHistory:
    if session_id not in store:
        store[session_id] = ChatMessageHistory()
    return store[session_id]

# 包装 Agent，添加记忆功能
agent_with_history = RunnableWithMessageHistory(
    agent_executor,
    get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history"
)

# 使用带记忆的 Agent
config = {"configurable": {"session_id": "user_123"}}

# 第一轮对话
response1 = agent_with_history.invoke(
    {"input": "我叫张三，今年 25 岁"},
    config=config
)

# 第二轮对话（Agent 会记住之前的信息）
response2 = agent_with_history.invoke(
    {"input": "你还记得我的名字吗？"},
    config=config
)
print(response2["output"])  # 会回答"张三"
```

---

## 第30章：LlamaIndex 实战

### 30.1 LlamaIndex 核心概念

```
LlamaIndex 专注于数据索引和检索

┌─────────────────────────────────────────────────────────┐
│                LlamaIndex 核心组件                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Data Connectors（数据连接器）                          │
│  ─────────────────────────────                          │
│  • 从各种来源加载数据                                  │
│  • 文件、数据库、API、网页等                           │
│                                                         │
│  Documents & Nodes（文档与节点）                        │
│  ───────────────────────────────                        │
│  • Document：原始文档                                  │
│  • Node：分块后的文档单元                              │
│  • 包含文本和元数据                                    │
│                                                         │
│  Indices（索引）                                        │
│  ──────────────                                         │
│  • VectorStoreIndex：向量索引                          │
│  • SummaryIndex：摘要索引                              │
│  • TreeIndex：树形索引                                 │
│  • KeywordTableIndex：关键词索引                       │
│                                                         │
│  Query Engines（查询引擎）                              │
│  ─────────────────────────                              │
│  • 处理用户查询                                        │
│  • 检索 + 生成                                         │
│                                                         │
│  Chat Engines（聊天引擎）                               │
│  ─────────────────────                                  │
│  • 多轮对话支持                                        │
│  • 上下文管理                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 30.2 快速开始

```python
# 安装
# pip install llama-index llama-index-llms-openai llama-index-embeddings-openai

import os
os.environ["OPENAI_API_KEY"] = "your-api-key"

from llama_index.core import VectorStoreIndex, SimpleDirectoryReader

# === 最简单的 RAG ===

# 1. 加载文档
documents = SimpleDirectoryReader("./data").load_data()

# 2. 创建索引（自动分块、向量化）
index = VectorStoreIndex.from_documents(documents)

# 3. 查询
query_engine = index.as_query_engine()
response = query_engine.query("这个项目的主要功能是什么？")
print(response)
```

### 30.3 自定义配置

```python
from llama_index.core import (
    VectorStoreIndex,
    SimpleDirectoryReader,
    Settings,
    StorageContext,
    load_index_from_storage
)
from llama_index.core.node_parser import SentenceSplitter
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# === 配置全局设置 ===
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0.1)
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")

# === 自定义文档处理 ===
# 加载文档
documents = SimpleDirectoryReader(
    input_dir="./data",
    recursive=True,  # 递归子目录
    required_exts=[".txt", ".md", ".pdf"],  # 指定文件类型
).load_data()

# 自定义分块
node_parser = SentenceSplitter(
    chunk_size=512,
    chunk_overlap=50,
)

nodes = node_parser.get_nodes_from_documents(documents)
print(f"生成了 {len(nodes)} 个节点")

# === 创建索引 ===
index = VectorStoreIndex(nodes)

# === 持久化存储 ===
# 保存
index.storage_context.persist(persist_dir="./storage")

# 加载
storage_context = StorageContext.from_defaults(persist_dir="./storage")
index = load_index_from_storage(storage_context)

# === 配置查询引擎 ===
query_engine = index.as_query_engine(
    similarity_top_k=5,  # 检索 top 5
    response_mode="compact",  # 响应模式
)
```

### 30.4 高级检索策略

```python
from llama_index.core import VectorStoreIndex, Settings
from llama_index.core.retrievers import VectorIndexRetriever
from llama_index.core.query_engine import RetrieverQueryEngine
from llama_index.core.postprocessor import (
    SimilarityPostprocessor,
    KeywordNodePostprocessor
)

# === 自定义检索器 ===
retriever = VectorIndexRetriever(
    index=index,
    similarity_top_k=10,  # 先检索多一些
)

# === 添加后处理器 ===
# 相似度过滤
similarity_filter = SimilarityPostprocessor(
    similarity_cutoff=0.7  # 过滤掉相似度低于 0.7 的
)

# 关键词过滤
keyword_filter = KeywordNodePostprocessor(
    required_keywords=["功能", "特性"],  # 必须包含的关键词
    exclude_keywords=["废弃", "过时"]    # 排除的关键词
)

# === 构建查询引擎 ===
query_engine = RetrieverQueryEngine(
    retriever=retriever,
    node_postprocessors=[similarity_filter, keyword_filter]
)

# === 混合检索 ===
from llama_index.core.retrievers import QueryFusionRetriever

# 融合多个检索器的结果
fusion_retriever = QueryFusionRetriever(
    retrievers=[retriever1, retriever2],
    similarity_top_k=5,
    num_queries=4,  # 生成多个查询变体
    mode="reciprocal_rerank",  # 融合排序方式
)
```

### 30.5 构建聊天引擎

```python
from llama_index.core.memory import ChatMemoryBuffer

# === 简单聊天引擎 ===
chat_engine = index.as_chat_engine(
    chat_mode="condense_plus_context",  # 模式
    verbose=True
)

# 多轮对话
response = chat_engine.chat("这个项目是做什么的？")
print(response)

response = chat_engine.chat("它有哪些主要功能？")
print(response)

response = chat_engine.chat("请详细解释第一个功能")
print(response)

# === 带记忆的聊天引擎 ===
memory = ChatMemoryBuffer.from_defaults(token_limit=3000)

chat_engine = index.as_chat_engine(
    chat_mode="context",
    memory=memory,
    system_prompt="""你是一个技术文档助手。
基于提供的文档回答问题。
如果不确定，请说明。
""",
)

# 重置对话
chat_engine.reset()
```

---

## 第31章：自定义 Agent 开发

### 31.1 从零构建简单 Agent

```python
import json
from openai import OpenAI

client = OpenAI()

# === 定义工具 ===
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_knowledge_base",
            "description": "在知识库中搜索信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {
                        "type": "string",
                        "description": "搜索关键词"
                    }
                },
                "required": ["query"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "execute_code",
            "description": "执行 Python 代码",
            "parameters": {
                "type": "object",
                "properties": {
                    "code": {
                        "type": "string",
                        "description": "要执行的 Python 代码"
                    }
                },
                "required": ["code"]
            }
        }
    }
]

# === 工具实现 ===
def search_knowledge_base(query: str) -> str:
    # 这里应该实现实际的搜索逻辑
    return f"搜索结果：关于 '{query}' 的信息..."

def execute_code(code: str) -> str:
    try:
        # 注意：生产环境需要沙箱！
        exec_globals = {}
        exec(code, exec_globals)
        return f"执行成功"
    except Exception as e:
        return f"执行错误：{str(e)}"

tool_functions = {
    "search_knowledge_base": search_knowledge_base,
    "execute_code": execute_code
}

# === Agent 主循环 ===
def run_agent(user_message: str, max_iterations: int = 10):
    messages = [
        {"role": "system", "content": "你是一个有用的助手，可以搜索知识库和执行代码。"},
        {"role": "user", "content": user_message}
    ]
    
    for i in range(max_iterations):
        # 调用 LLM
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=messages,
            tools=tools,
            tool_choice="auto"
        )
        
        message = response.choices[0].message
        messages.append(message)
        
        # 检查是否需要调用工具
        if message.tool_calls:
            for tool_call in message.tool_calls:
                function_name = tool_call.function.name
                arguments = json.loads(tool_call.function.arguments)
                
                print(f"调用工具：{function_name}")
                print(f"参数：{arguments}")
                
                # 执行工具
                result = tool_functions[function_name](**arguments)
                
                # 将结果添加到消息
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })
        else:
            # 没有工具调用，返回最终回答
            return message.content
    
    return "达到最大迭代次数"

# === 使用 ===
result = run_agent("帮我搜索一下 Python 装饰器的使用方法")
print(result)
```

### 31.2 ReAct Agent 实现

```python
from openai import OpenAI
import re

client = OpenAI()

# ReAct 提示模板
REACT_PROMPT = """你是一个使用 ReAct 模式的 AI 助手。

对于每个问题，你应该按以下格式思考和行动：

Thought: 我需要思考这个问题...
Action: 工具名称[参数]
Observation: 工具返回的结果
... (重复 Thought/Action/Observation)
Thought: 我现在知道答案了
Final Answer: 最终答案

可用的工具：
- search[query]: 搜索信息
- calculate[expression]: 计算数学表达式
- lookup[term]: 查找术语定义

现在回答以下问题：
{question}
"""

def search(query: str) -> str:
    return f"搜索 '{query}' 的结果：..."

def calculate(expression: str) -> str:
    try:
        return str(eval(expression))
    except:
        return "计算错误"

def lookup(term: str) -> str:
    return f"'{term}' 的定义：..."

tools = {
    "search": search,
    "calculate": calculate,
    "lookup": lookup
}

def parse_action(text: str):
    """解析 Action: 工具名称[参数]"""
    match = re.search(r'Action:\s*(\w+)\[(.*?)\]', text)
    if match:
        return match.group(1), match.group(2)
    return None, None

def run_react_agent(question: str, max_steps: int = 10):
    prompt = REACT_PROMPT.format(question=question)
    full_response = ""
    
    for step in range(max_steps):
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{"role": "user", "content": prompt + full_response}],
            stop=["Observation:"],  # 在 Observation 处停止
            temperature=0
        )
        
        text = response.choices[0].message.content
        full_response += text
        
        # 检查是否有最终答案
        if "Final Answer:" in text:
            match = re.search(r'Final Answer:\s*(.*)', text, re.DOTALL)
            if match:
                return match.group(1).strip()
        
        # 解析并执行动作
        action, param = parse_action(text)
        if action and action in tools:
            observation = tools[action](param)
            full_response += f"\nObservation: {observation}\n"
        else:
            full_response += "\nObservation: 无法识别的动作\n"
    
    return "无法得出答案"

# 使用
answer = run_react_agent("计算 123 * 456，然后搜索这个数字的含义")
print(answer)
```

### 31.3 工具设计最佳实践

```python
from pydantic import BaseModel, Field
from typing import Optional, Literal
import json

# === 使用 Pydantic 定义工具参数 ===
class SearchParams(BaseModel):
    """知识库搜索参数"""
    query: str = Field(description="搜索关键词")
    top_k: int = Field(default=5, description="返回结果数量")
    filter_type: Optional[Literal["docs", "code", "all"]] = Field(
        default="all",
        description="过滤类型"
    )

class FileOperationParams(BaseModel):
    """文件操作参数"""
    path: str = Field(description="文件路径")
    operation: Literal["read", "write", "append"] = Field(
        description="操作类型"
    )
    content: Optional[str] = Field(default=None, description="写入内容")

# === 工具装饰器 ===
def tool(name: str, description: str, params_model: type):
    """工具装饰器，自动生成 OpenAI function 格式"""
    def decorator(func):
        func._tool_name = name
        func._tool_description = description
        func._tool_schema = params_model.model_json_schema()
        
        def wrapper(**kwargs):
            # 验证参数
            params = params_model(**kwargs)
            return func(params)
        
        wrapper._tool_definition = {
            "type": "function",
            "function": {
                "name": name,
                "description": description,
                "parameters": params_model.model_json_schema()
            }
        }
        
        return wrapper
    return decorator

# === 使用装饰器定义工具 ===
@tool("search", "在知识库中搜索", SearchParams)
def search_tool(params: SearchParams) -> str:
    return f"搜索 '{params.query}'，返回 top {params.top_k} 结果"

@tool("file_op", "文件操作", FileOperationParams)
def file_operation(params: FileOperationParams) -> str:
    if params.operation == "read":
        return f"读取文件：{params.path}"
    elif params.operation == "write":
        return f"写入文件：{params.path}"
    else:
        return f"追加文件：{params.path}"

# 获取所有工具定义
all_tools = [search_tool._tool_definition, file_operation._tool_definition]
print(json.dumps(all_tools, indent=2, ensure_ascii=False))
```

---

## 本章小结

```
核心要点回顾：

1. 框架选择
   ├── LangChain：全能型，生态丰富
   ├── LlamaIndex：RAG 专精，数据处理强
   ├── OpenAI Assistants：托管服务，简单上手
   └── 根据需求选择合适的框架

2. LangChain 实战
   ├── LCEL 语法：管道符构建链
   ├── RAG：文档加载、分块、向量化、检索
   ├── Agent：工具定义、执行器
   └── 记忆：ChatMessageHistory

3. LlamaIndex 实战
   ├── 核心：Documents → Nodes → Index
   ├── 索引类型：Vector、Summary、Tree 等
   ├── 查询/聊天引擎
   └── 高级检索：后处理、混合检索

4. 自定义 Agent
   ├── 工具调用机制
   ├── ReAct 模式
   └── 工具设计最佳实践
```

---

**下一章 → [第32章：RAG 高级技术](./10-rag-advanced.md)**
