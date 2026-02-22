# 进阶模块五：多 Agent 框架

> 构建协作式 AI 系统

---

## 第39章：多 Agent 框架概览

### 39.1 为什么需要多 Agent

```
单 Agent 的局限性：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  复杂任务示例："开发一个 Web 应用"                     │
│                                                         │
│  单 Agent 的问题：                                      │
│  ─────────────────                                      │
│  • 上下文过长，容易遗忘                                │
│  • 需要多种专业能力                                    │
│  • 难以并行处理                                        │
│  • 错误难以隔离                                        │
│                                                         │
│  多 Agent 的优势：                                      │
│  ─────────────────                                      │
│  • 专业分工                                            │
│  • 并行执行                                            │
│  • 错误隔离                                            │
│  • 更好的可扩展性                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘

多 Agent 协作模式：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  1. 顺序执行（Pipeline）                                │
│  ─────────────────────                                  │
│  Agent A → Agent B → Agent C → 结果                    │
│                                                         │
│  适用：任务有明确的阶段                                │
│  示例：需求分析 → 代码编写 → 代码审查                 │
│                                                         │
│  2. 并行执行                                            │
│  ───────────                                            │
│       ┌─ Agent A ─┐                                    │
│  任务 ┼─ Agent B ─┼→ 合并结果                          │
│       └─ Agent C ─┘                                    │
│                                                         │
│  适用：任务可分解且独立                                │
│  示例：同时搜索多个来源                                │
│                                                         │
│  3. 讨论/辩论                                           │
│  ───────────                                            │
│  Agent A ←→ Agent B ←→ Agent C                         │
│        多轮交互，达成共识                               │
│                                                         │
│  适用：需要多角度思考的问题                            │
│  示例：方案评审、风险分析                              │
│                                                         │
│  4. 层级管理                                            │
│  ───────────                                            │
│           Manager Agent                                 │
│          /     |     \                                  │
│     Worker  Worker  Worker                             │
│                                                         │
│  适用：复杂项目管理                                    │
│  示例：软件开发项目                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 39.2 主流多 Agent 框架

```
┌─────────────────────────────────────────────────────────┐
│                  多 Agent 框架对比                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  LangGraph                                              │
│  ─────────                                              │
│  • LangChain 团队出品                                  │
│  • 基于图的状态机                                      │
│  • 灵活的流程控制                                      │
│  • 与 LangChain 生态集成                               │
│  • 适合自定义复杂流程                                  │
│                                                         │
│  AutoGen（微软）                                        │
│  ────────────────                                       │
│  • 多 Agent 对话框架                                   │
│  • 内置多种 Agent 类型                                 │
│  • 人机协作支持                                        │
│  • 代码执行能力强                                      │
│  • 适合研究和原型                                      │
│                                                         │
│  CrewAI                                                 │
│  ───────                                                │
│  • 角色扮演导向                                        │
│  • 简单易用                                            │
│  • 任务分配明确                                        │
│  • 适合快速构建团队协作场景                            │
│                                                         │
│  MetaGPT                                                │
│  ────────                                               │
│  • 软件开发专用                                        │
│  • 模拟软件公司                                        │
│  • 从需求到代码全流程                                  │
│                                                         │
│  选择建议：                                             │
│  ┌────────────────────────────────────────────────┐    │
│  │ 需求                    │ 推荐                 │    │
│  ├────────────────────────────────────────────────┤    │
│  │ 复杂流程控制            │ LangGraph           │    │
│  │ 多 Agent 对话           │ AutoGen             │    │
│  │ 快速原型                │ CrewAI              │    │
│  │ 软件开发                │ MetaGPT             │    │
│  │ 与 LangChain 集成       │ LangGraph           │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 第40章：LangGraph 实战

### 40.1 LangGraph 核心概念

```
LangGraph 使用图来定义 Agent 流程

核心概念：

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  State（状态）                                          │
│  ─────────────                                          │
│  • 在图中流转的数据                                    │
│  • 所有节点共享和修改                                  │
│  • 使用 TypedDict 定义                                 │
│                                                         │
│  Node（节点）                                           │
│  ────────────                                           │
│  • 执行具体操作的函数                                  │
│  • 接收 State，返回修改                                │
│  • 可以是 Agent、工具调用、条件判断等                 │
│                                                         │
│  Edge（边）                                             │
│  ──────────                                             │
│  • 连接节点，定义流转路径                              │
│  • 普通边：固定流向                                    │
│  • 条件边：根据条件选择下一个节点                      │
│                                                         │
│  Graph（图）                                            │
│  ──────────                                             │
│  • 由节点和边组成                                      │
│  • 定义完整的执行流程                                  │
│  • 编译后可执行                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 40.2 简单示例：顺序执行

```python
# pip install langgraph langchain-openai

from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langchain_openai import ChatOpenAI

# === 1. 定义状态 ===
class State(TypedDict):
    topic: str           # 用户输入的主题
    outline: str         # 大纲
    draft: str           # 初稿
    final: str           # 终稿

# === 2. 定义节点函数 ===
llm = ChatOpenAI(model="gpt-4o-mini")

def create_outline(state: State) -> dict:
    """创建大纲"""
    response = llm.invoke(f"为以下主题创建一个简短的文章大纲：{state['topic']}")
    return {"outline": response.content}

def write_draft(state: State) -> dict:
    """写初稿"""
    response = llm.invoke(f"根据以下大纲写一篇文章初稿：\n{state['outline']}")
    return {"draft": response.content}

def polish_article(state: State) -> dict:
    """润色文章"""
    response = llm.invoke(f"润色以下文章，改进表达：\n{state['draft']}")
    return {"final": response.content}

# === 3. 构建图 ===
workflow = StateGraph(State)

# 添加节点
workflow.add_node("outline", create_outline)
workflow.add_node("draft", write_draft)
workflow.add_node("polish", polish_article)

# 添加边
workflow.set_entry_point("outline")
workflow.add_edge("outline", "draft")
workflow.add_edge("draft", "polish")
workflow.add_edge("polish", END)

# === 4. 编译并执行 ===
app = workflow.compile()

# 执行
result = app.invoke({"topic": "人工智能的未来发展"})
print(result["final"])
```

### 40.3 条件分支

```python
from typing import Literal

class State(TypedDict):
    query: str
    category: str
    response: str

def classify_query(state: State) -> dict:
    """分类用户查询"""
    response = llm.invoke(
        f"将以下查询分类为 'technical' 或 'general'：\n{state['query']}\n只返回分类词。"
    )
    return {"category": response.content.strip().lower()}

def handle_technical(state: State) -> dict:
    """处理技术问题"""
    response = llm.invoke(f"作为技术专家回答：{state['query']}")
    return {"response": response.content}

def handle_general(state: State) -> dict:
    """处理一般问题"""
    response = llm.invoke(f"友好地回答：{state['query']}")
    return {"response": response.content}

def route_query(state: State) -> Literal["technical", "general"]:
    """路由函数"""
    if state["category"] == "technical":
        return "technical"
    return "general"

# 构建图
workflow = StateGraph(State)

workflow.add_node("classify", classify_query)
workflow.add_node("technical", handle_technical)
workflow.add_node("general", handle_general)

workflow.set_entry_point("classify")

# 条件边
workflow.add_conditional_edges(
    "classify",
    route_query,
    {
        "technical": "technical",
        "general": "general"
    }
)

workflow.add_edge("technical", END)
workflow.add_edge("general", END)

app = workflow.compile()
```

### 40.4 循环与迭代

```python
from typing import List

class State(TypedDict):
    task: str
    plan: List[str]
    current_step: int
    results: List[str]
    final_answer: str

def create_plan(state: State) -> dict:
    """创建执行计划"""
    response = llm.invoke(
        f"为以下任务创建执行步骤（每行一个步骤）：\n{state['task']}"
    )
    steps = [s.strip() for s in response.content.split("\n") if s.strip()]
    return {"plan": steps, "current_step": 0, "results": []}

def execute_step(state: State) -> dict:
    """执行当前步骤"""
    current = state["current_step"]
    step = state["plan"][current]
    
    context = "\n".join(state["results"]) if state["results"] else "无"
    response = llm.invoke(
        f"执行以下步骤：{step}\n之前的结果：{context}"
    )
    
    new_results = state["results"] + [f"步骤 {current+1}: {response.content}"]
    return {
        "results": new_results,
        "current_step": current + 1
    }

def summarize(state: State) -> dict:
    """总结所有结果"""
    all_results = "\n".join(state["results"])
    response = llm.invoke(f"总结以下执行结果：\n{all_results}")
    return {"final_answer": response.content}

def should_continue(state: State) -> Literal["continue", "summarize"]:
    """判断是否继续"""
    if state["current_step"] < len(state["plan"]):
        return "continue"
    return "summarize"

# 构建图
workflow = StateGraph(State)

workflow.add_node("plan", create_plan)
workflow.add_node("execute", execute_step)
workflow.add_node("summarize", summarize)

workflow.set_entry_point("plan")
workflow.add_edge("plan", "execute")

# 循环边
workflow.add_conditional_edges(
    "execute",
    should_continue,
    {
        "continue": "execute",
        "summarize": "summarize"
    }
)

workflow.add_edge("summarize", END)

app = workflow.compile()
```

---

## 第41章：AutoGen 实战

### 41.1 AutoGen 基础

```python
# pip install pyautogen

import autogen
from autogen import AssistantAgent, UserProxyAgent, GroupChat, GroupChatManager

# 配置 LLM
config_list = [{
    "model": "gpt-4o-mini",
    "api_key": "your-api-key"
}]

llm_config = {
    "config_list": config_list,
    "temperature": 0.7
}

# === 基础：两个 Agent 对话 ===

# 创建助手 Agent
assistant = AssistantAgent(
    name="助手",
    system_message="你是一个有用的 AI 助手。",
    llm_config=llm_config
)

# 创建用户代理（可以执行代码）
user_proxy = UserProxyAgent(
    name="用户",
    human_input_mode="NEVER",  # 不需要人工输入
    max_consecutive_auto_reply=10,
    code_execution_config={
        "work_dir": "coding",
        "use_docker": False
    }
)

# 开始对话
user_proxy.initiate_chat(
    assistant,
    message="用 Python 写一个函数计算斐波那契数列"
)
```

### 41.2 多 Agent 群聊

```python
# === 创建多个专门的 Agent ===

# 产品经理
pm = AssistantAgent(
    name="产品经理",
    system_message="""你是一个产品经理。
你的职责是：
- 理解用户需求
- 编写需求文档
- 协调团队工作
当需求明确后，将任务交给开发人员。""",
    llm_config=llm_config
)

# 开发人员
developer = AssistantAgent(
    name="开发人员",
    system_message="""你是一个高级开发人员。
你的职责是：
- 编写高质量代码
- 实现产品需求
- 遵循最佳实践
编写完代码后，请求代码审查。""",
    llm_config=llm_config
)

# 代码审查员
reviewer = AssistantAgent(
    name="审查员",
    system_message="""你是一个代码审查专家。
你的职责是：
- 审查代码质量
- 指出潜在问题
- 提出改进建议
审查通过后说"APPROVED"。""",
    llm_config=llm_config
)

# 用户代理
user = UserProxyAgent(
    name="用户",
    human_input_mode="TERMINATE",  # 遇到 TERMINATE 停止
    max_consecutive_auto_reply=20,
    code_execution_config={"work_dir": "project"}
)

# === 创建群聊 ===
groupchat = GroupChat(
    agents=[user, pm, developer, reviewer],
    messages=[],
    max_round=20
)

# 群聊管理器
manager = GroupChatManager(
    groupchat=groupchat,
    llm_config=llm_config
)

# 开始群聊
user.initiate_chat(
    manager,
    message="我需要一个简单的待办事项 API，支持增删改查"
)
```

### 41.3 自定义对话流程

```python
# === 自定义 Agent 选择逻辑 ===

def custom_speaker_selection(
    last_speaker,
    groupchat
) -> AssistantAgent:
    """自定义发言者选择"""
    messages = groupchat.messages
    
    if len(messages) <= 1:
        return pm  # 开始时让 PM 分析需求
    
    last_message = messages[-1]["content"].lower()
    
    # 根据消息内容决定下一个发言者
    if "需求" in last_message or "功能" in last_message:
        return developer
    elif "代码" in last_message or "实现" in last_message:
        return reviewer
    elif "approved" in last_message:
        return user
    
    return None  # 让系统自动选择

groupchat = GroupChat(
    agents=[user, pm, developer, reviewer],
    messages=[],
    max_round=20,
    speaker_selection_method=custom_speaker_selection  # 使用自定义选择
)
```

---

## 第42章：CrewAI 实战

### 42.1 CrewAI 基础

```python
# pip install crewai crewai-tools

from crewai import Agent, Task, Crew, Process
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4o-mini")

# === 定义 Agent（角色）===

researcher = Agent(
    role="研究员",
    goal="深入研究给定主题，收集全面的信息",
    backstory="""你是一个经验丰富的研究员，擅长从多个来源
收集和整理信息。你注重细节，能够识别关键信息。""",
    llm=llm,
    verbose=True
)

writer = Agent(
    role="技术作家",
    goal="将研究结果转化为清晰、易懂的文章",
    backstory="""你是一个专业的技术作家，擅长将复杂的技术概念
用简单的语言解释清楚。你的文章结构清晰，逻辑严密。""",
    llm=llm,
    verbose=True
)

editor = Agent(
    role="编辑",
    goal="确保文章质量，提升可读性",
    backstory="""你是一个资深编辑，有多年的审稿经验。
你擅长发现文章中的问题并提出改进建议。""",
    llm=llm,
    verbose=True
)
```

### 42.2 定义任务

```python
# === 定义任务 ===

research_task = Task(
    description="""研究以下主题：{topic}
    
    你需要：
    1. 收集该主题的核心概念
    2. 找出最新的发展动态
    3. 识别关键趋势和挑战
    4. 整理出 3-5 个关键要点
    """,
    expected_output="一份详细的研究报告，包含关键发现和数据",
    agent=researcher
)

writing_task = Task(
    description="""基于研究报告，撰写一篇文章
    
    要求：
    1. 标题吸引人
    2. 结构清晰（引言、正文、结论）
    3. 语言简洁易懂
    4. 包含具体例子
    5. 800-1200 字
    """,
    expected_output="一篇结构完整、内容充实的文章",
    agent=writer,
    context=[research_task]  # 依赖研究任务的结果
)

editing_task = Task(
    description="""审核并改进文章
    
    检查：
    1. 语法和拼写
    2. 逻辑连贯性
    3. 表达是否清晰
    4. 是否需要补充内容
    
    提供具体的修改建议和最终版本
    """,
    expected_output="修改后的最终文章和修改说明",
    agent=editor,
    context=[writing_task]
)
```

### 42.3 创建并运行 Crew

```python
# === 创建 Crew ===

content_crew = Crew(
    agents=[researcher, writer, editor],
    tasks=[research_task, writing_task, editing_task],
    process=Process.sequential,  # 顺序执行
    verbose=True
)

# === 运行 ===
result = content_crew.kickoff(inputs={"topic": "2025年人工智能发展趋势"})
print(result)
```

### 42.4 并行执行

```python
from crewai import Crew, Process

# 定义可以并行的任务
parallel_tasks = [
    Task(
        description="分析市场数据",
        expected_output="市场分析报告",
        agent=market_analyst
    ),
    Task(
        description="分析技术趋势",
        expected_output="技术趋势报告",
        agent=tech_analyst
    ),
    Task(
        description="分析竞争对手",
        expected_output="竞争分析报告",
        agent=competitor_analyst
    )
]

# 汇总任务
summary_task = Task(
    description="整合所有分析报告，生成综合报告",
    expected_output="综合分析报告",
    agent=strategist,
    context=parallel_tasks  # 等待所有并行任务完成
)

crew = Crew(
    agents=[market_analyst, tech_analyst, competitor_analyst, strategist],
    tasks=parallel_tasks + [summary_task],
    process=Process.hierarchical,  # 层级处理
    manager_llm=llm
)

result = crew.kickoff()
```

---

## 第43章：多 Agent 最佳实践

### 43.1 Agent 设计原则

```
┌─────────────────────────────────────────────────────────┐
│                 Agent 设计原则                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 单一职责                                            │
│  ───────────                                            │
│  每个 Agent 专注于一个领域/任务                        │
│                                                         │
│  ❌ 不好：一个 Agent 同时做研究、写作、审核            │
│  ✅ 好：分成研究员、作家、审核员三个 Agent             │
│                                                         │
│  2. 明确的角色定义                                      │
│  ─────────────────                                      │
│  清晰描述每个 Agent 的：                               │
│  • 角色/身份                                           │
│  • 目标                                                │
│  • 能力边界                                            │
│  • 行为准则                                            │
│                                                         │
│  3. 合理的通信协议                                      │
│  ─────────────────                                      │
│  定义 Agent 之间如何：                                 │
│  • 传递信息                                            │
│  • 请求协助                                            │
│  • 报告结果                                            │
│  • 处理冲突                                            │
│                                                         │
│  4. 适当的自主性                                        │
│  ─────────────                                          │
│  • 给 Agent 足够的决策空间                             │
│  • 但要设置边界和检查点                                │
│  • 关键操作需要确认                                    │
│                                                         │
│  5. 可观测性                                            │
│  ───────────                                            │
│  • 记录 Agent 的决策过程                               │
│  • 便于调试和优化                                      │
│  • 支持人工介入                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 43.2 常见问题与解决

```
┌─────────────────────────────────────────────────────────┐
│               多 Agent 常见问题                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  问题 1：无限循环                                       │
│  ─────────────────                                      │
│  Agent 之间互相推诿，无法终止                          │
│                                                         │
│  解决：                                                 │
│  • 设置最大轮次                                        │
│  • 明确终止条件                                        │
│  • 添加仲裁者角色                                      │
│                                                         │
│  问题 2：上下文丢失                                     │
│  ─────────────────                                      │
│  信息在传递过程中丢失或变形                            │
│                                                         │
│  解决：                                                 │
│  • 使用共享状态                                        │
│  • 结构化信息传递                                      │
│  • 关键信息重复确认                                    │
│                                                         │
│  问题 3：协调失败                                       │
│  ─────────────────                                      │
│  Agent 之间意见不一致                                  │
│                                                         │
│  解决：                                                 │
│  • 设立决策者角色                                      │
│  • 定义优先级规则                                      │
│  • 投票机制                                            │
│                                                         │
│  问题 4：成本过高                                       │
│  ─────────────────                                      │
│  多 Agent 消耗大量 Token                               │
│                                                         │
│  解决：                                                 │
│  • 简化不必要的对话                                    │
│  • 缓存重复计算                                        │
│  • 使用较小的模型处理简单任务                          │
│                                                         │
│  问题 5：质量不稳定                                     │
│  ─────────────────                                      │
│  输出质量波动大                                        │
│                                                         │
│  解决：                                                 │
│  • 添加质量检查节点                                    │
│  • 允许重试                                            │
│  • 降低 temperature                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 43.3 调试与监控

```python
import logging
from datetime import datetime

# === 日志系统 ===
class AgentLogger:
    def __init__(self, log_file: str = "agent_logs.jsonl"):
        self.log_file = log_file
        self.logs = []
    
    def log_message(self, 
                   agent_name: str, 
                   message_type: str,
                   content: str,
                   metadata: dict = None):
        """记录 Agent 消息"""
        log_entry = {
            "timestamp": datetime.now().isoformat(),
            "agent": agent_name,
            "type": message_type,
            "content": content,
            "metadata": metadata or {}
        }
        self.logs.append(log_entry)
        
        # 写入文件
        with open(self.log_file, "a") as f:
            import json
            f.write(json.dumps(log_entry, ensure_ascii=False) + "\n")
    
    def log_decision(self, agent_name: str, decision: str, reasoning: str):
        """记录决策"""
        self.log_message(
            agent_name,
            "decision",
            decision,
            {"reasoning": reasoning}
        )
    
    def log_error(self, agent_name: str, error: str, context: dict = None):
        """记录错误"""
        self.log_message(
            agent_name,
            "error",
            error,
            {"context": context}
        )
    
    def get_conversation_trace(self) -> list:
        """获取对话轨迹"""
        return [
            f"[{log['timestamp']}] {log['agent']}: {log['content'][:100]}..."
            for log in self.logs
        ]


# === 成本追踪 ===
class CostTracker:
    def __init__(self):
        self.total_tokens = {"input": 0, "output": 0}
        self.by_agent = {}
    
    def add_usage(self, agent_name: str, input_tokens: int, output_tokens: int):
        """记录 Token 使用"""
        self.total_tokens["input"] += input_tokens
        self.total_tokens["output"] += output_tokens
        
        if agent_name not in self.by_agent:
            self.by_agent[agent_name] = {"input": 0, "output": 0}
        self.by_agent[agent_name]["input"] += input_tokens
        self.by_agent[agent_name]["output"] += output_tokens
    
    def get_cost_estimate(self, 
                         input_price: float = 0.001,   # per 1K tokens
                         output_price: float = 0.002) -> dict:
        """估算成本"""
        input_cost = (self.total_tokens["input"] / 1000) * input_price
        output_cost = (self.total_tokens["output"] / 1000) * output_price
        return {
            "total_cost": input_cost + output_cost,
            "input_cost": input_cost,
            "output_cost": output_cost,
            "total_tokens": sum(self.total_tokens.values())
        }


# 使用示例
logger = AgentLogger()
cost_tracker = CostTracker()

# 在 Agent 执行中
logger.log_message("研究员", "思考", "正在分析用户需求...")
logger.log_decision("研究员", "开始网络搜索", "需要更多背景信息")
cost_tracker.add_usage("研究员", input_tokens=500, output_tokens=200)
```

---

## 本章小结

```
核心要点回顾：

1. 多 Agent 概念
   ├── 解决单 Agent 的局限
   ├── 协作模式：顺序、并行、讨论、层级
   └── 主流框架：LangGraph、AutoGen、CrewAI

2. LangGraph
   ├── 基于图的状态机
   ├── 节点、边、状态
   ├── 条件分支和循环
   └── 灵活的流程控制

3. AutoGen
   ├── 多 Agent 对话
   ├── 群聊和管理器
   ├── 代码执行能力
   └── 人机协作支持

4. CrewAI
   ├── 角色扮演导向
   ├── 任务和依赖
   ├── 顺序/并行执行
   └── 快速构建团队

5. 最佳实践
   ├── Agent 设计原则
   ├── 常见问题处理
   └── 调试与监控
```

---

**返回 → [课程总览](./00-overview.md)**
