# 阶段 2: LangGraph 图结构 - 核心逻辑

## 📚 学习目标

- 理解 LangGraph 的工作原理
- 掌握节点、边、条件路由的概念
- 理解并行执行和 Send 机制
- 深入理解每个节点的实现
- 掌握状态转换流程

---

## 🎯 LangGraph 核心概念

### 图 (Graph) 的构成

LangGraph 使用**有向图**来编排 AI Agent 的工作流程:

```
节点 (Node) → 执行任务的函数
边 (Edge) → 连接节点的路径
状态 (State) → 在节点间流动的数据
```

### StateGraph 类

**文件位置**: `backend/src/agent/graph.py:269`

```python
builder = StateGraph(OverallState, config_schema=Configuration)
```

**参数说明**:
- `OverallState`: 图的状态类型,定义节点间传递的数据结构
- `config_schema`: 配置类型,用于从环境变量或 RunnableConfig 加载配置

---

## 📊 完整图结构

```
┌─────────────────────────────────────────────────────────────┐
│                        START                                │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                  generate_query                             │
│  使用 Gemini 2.0 Flash 生成初始搜索查询                      │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│              continue_to_web_research                       │
│     条件边: 为每个查询创建并行分支 (使用 Send)                │
└─────────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
    ┌──────────┐    ┌──────────┐    ┌──────────┐
    │   web_   │    │   web_   │    │   web_   │
    │ research │    │ research │    │ research │
    │  (查询1) │    │  (查询2) │    │  (查询3) │
    └──────────┘    └──────────┘    └──────────┘
           │               │               │
           └───────────────┼───────────────┘
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     reflection                              │
│  分析搜索结果,识别知识缺口,生成后续查询                        │
└─────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                 evaluate_research                           │
│  条件路由: 判断研究是否充分或达到最大循环次数                  │
└─────────────────────────────────────────────────────────────┘
                    │                            │
            研究不充分                       研究充分
            且未达最大循环                     或达最大循环
                    │                            │
                    ▼                            ▼
         ┌──────────────────┐        ┌─────────────────────┐
         │  web_research    │        │  finalize_answer    │
         │  (后续查询)       │        │  生成最终答案        │
         └──────────────────┘        └─────────────────────┘
                                             │
                                             ▼
                                     ┌───────────────┐
                                     │      END      │
                                     └───────────────┘
```

---

## 🔧 节点详解

### 1. generate_query 节点

**文件位置**: `backend/src/agent/graph.py:44-81`

#### 功能
根据用户的问题生成优化的搜索查询

#### 实现细节

```python
def generate_query(state: OverallState, config: RunnableConfig) -> QueryGenerationState:
    configurable = Configuration.from_runnable_config(config)

    # 检查并设置初始查询数量
    if state.get("initial_search_query_count") is None:
        state["initial_search_query_count"] = configurable.number_of_initial_queries

    # 初始化 Gemini 2.0 Flash
    llm = ChatGoogleGenerativeAI(
        model=configurable.query_generator_model,  # gemini-2.0-flash
        temperature=1.0,  # 高温度以增加多样性
        max_retries=2,
        api_key=os.getenv("GEMINI_API_KEY"),
    )
    structured_llm = llm.with_structured_output(SearchQueryList)

    # 格式化提示词
    formatted_prompt = query_writer_instructions.format(
        current_date=get_current_date(),
        research_topic=get_research_topic(state["messages"]),
        number_queries=state["initial_search_query_count"],
    )

    # 生成搜索查询
    result = structured_llm.invoke(formatted_prompt)
    return {"search_query": result.query}
```

#### 关键点

| 元素 | 值 | 说明 |
|------|-----|------|
| 输入状态 | `OverallState` | 包含用户消息 |
| 输出状态 | `QueryGenerationState` | 返回 search_query |
| 模型 | `gemini-2.0-flash` | 快速生成,适合查询生成 |
| Temperature | `1.0` | 高温度增加查询多样性 |
| 结构化输出 | `SearchQueryList` | 确保输出格式正确 |

#### 提示词分析

**query_writer_instructions** (prompts.py:9-34):

```
- 优先使用单一查询
- 每个查询关注一个特定方面
- 最多生成 number_queries 个查询
- 查询应该多样化
- 不要生成相似的查询
- 确保获取最新信息
```

#### 输出示例

```json
{
  "query": [
    "renewable energy trends 2024",
    "solar energy advancements 2024",
    "wind energy technology 2024"
  ],
  "rationale": "These queries cover recent developments in different renewable energy sectors."
}
```

---

### 2. continue_to_web_research 函数

**文件位置**: `backend/src/agent/graph.py:84-92`

#### 功能
将查询分发到多个并行的 web_research 节点

#### 实现细节

```python
def continue_to_web_research(state: QueryGenerationState):
    """为每个搜索查询创建一个 web_research 任务"""
    return [
        Send("web_research", {"search_query": search_query, "id": int(idx)})
        for idx, search_query in enumerate(state["search_query"])
    ]
```

#### Send 对象详解

`Send` 是 LangGraph 中实现**并行执行**的关键机制:

```python
Send("目标节点", {状态更新})
```

**参数说明**:
- 第一个参数: 目标节点名称
- 第二个参数: 传递给该节点的状态更新

#### 并行执行示例

假设生成了 3 个查询:

```python
state["search_query"] = [
    "renewable energy trends 2024",
    "solar energy advancements",
    "wind energy technology"
]
```

`continue_to_web_research` 返回:

```python
[
    Send("web_research", {"search_query": "renewable energy trends 2024", "id": 0}),
    Send("web_research", {"search_query": "solar energy advancements", "id": 1}),
    Send("web_research", {"search_query": "wind energy technology", "id": 2})
]
```

LangGraph 会创建 **3 个并行任务**,每个任务独立执行 `web_research` 节点。

#### 为什么需要 continue_to_web_research?

✅ **单一职责原则**: `generate_query` 只负责生成查询,不负责分发
✅ **并行优化**: 多个查询可以同时执行,提高效率
✅ **状态隔离**: 每个搜索任务有自己的 ID,便于追踪

---

### 3. web_research 节点

**文件位置**: `backend/src/agent/graph.py:95-136`

#### 功能
使用 Google Search API 执行网络搜索并生成摘要

#### 实现细节

```python
def web_research(state: WebSearchState, config: RunnableConfig) -> OverallState:
    configurable = Configuration.from_runnable_config(config)

    # 格式化提示词
    formatted_prompt = web_searcher_instructions.format(
        current_date=get_current_date(),
        research_topic=state["search_query"],
    )

    # 使用 Google GenAI Client (非 LangChain wrapper)
    # 因为 LangChain 客户端不返回 grounding_metadata
    response = genai_client.models.generate_content(
        model=configurable.query_generator_model,
        contents=formatted_prompt,
        config={
            "tools": [{"google_search": {}}],  # 启用 Google Search
            "temperature": 0,
        },
    )

    # 解析 URL 为短 URL (节省 token)
    resolved_urls = resolve_urls(
        response.candidates[0].grounding_metadata.grounding_chunks,
        state["id"]
    )

    # 提取引用并插入到文本中
    citations = get_citations(response, resolved_urls)
    modified_text = insert_citation_markers(response.text, citations)
    sources_gathered = [item for citation in citations for item in citation["segments"]]

    return {
        "sources_gathered": sources_gathered,
        "search_query": [state["search_query"]],
        "web_research_result": [modified_text],
    }
```

#### 关键点

| 元素 | 值 | 说明 |
|------|-----|------|
| 输入状态 | `WebSearchState` | 单个查询和 ID |
| 输出状态 | `OverallState` | 累积搜索结果和来源 |
| API | Google GenAI Client | 原生客户端,支持 grounding |
| Tools | `google_search` | 启用 Google 搜索 |
| Temperature | `0` | 低温度,确保准确性 |

#### 为什么使用原生 Google GenAI Client?

```python
# ❌ LangChain 客户端
llm = ChatGoogleGenerativeAI(...)
response = llm.invoke(...)  # 不返回 grounding_metadata

# ✅ 原生客户端
from google.genai import Client
client = Client(...)
response = client.models.generate_content(...)  # 返回 grounding_metadata
```

`grounding_metadata` 包含:
- `grounding_chunks`: 搜索结果的详细信息
- `grounding_supports`: 引用的位置和索引
- 这些信息用于生成带引用的答案

#### 引用处理流程

```
Google Search API
    ↓
grounding_metadata
    ├─ grounding_chunks (来源信息)
    └─ grounding_supports (引用位置)
    ↓
resolve_urls (转换为短 URL)
    ↓
get_citations (提取引用对象)
    ↓
insert_citation_markers (插入引用标记)
    ↓
最终文本 (带引用的 Markdown)
```

#### 输出示例

```markdown
Renewable energy saw significant growth in 2024 [1](https://vertexaisearch.cloud.google.com/id/0-0).
Solar capacity increased by 25% compared to 2023 [2](https://vertexaisearch.cloud.google.com/id/0-1).
```

---

### 4. reflection 节点

**文件位置**: `backend/src/agent/graph.py:139-180`

#### 功能
分析搜索结果,识别知识缺口,生成后续查询

#### 实现细节

```python
def reflection(state: OverallState, config: RunnableConfig) -> ReflectionState:
    configurable = Configuration.from_runnable_config(config)

    # 增加研究循环计数
    state["research_loop_count"] = state.get("research_loop_count", 0) + 1
    reasoning_model = state.get("reasoning_model", configurable.reflection_model)

    # 格式化提示词
    formatted_prompt = reflection_instructions.format(
        current_date=get_current_date(),
        research_topic=get_research_topic(state["messages"]),
        summaries="\n\n---\n\n".join(state["web_research_result"]),  # 所有搜索结果
    )

    # 初始化推理模型
    llm = ChatGoogleGenerativeAI(
        model=reasoning_model,  # gemini-2.5-flash
        temperature=1.0,
        max_retries=2,
        api_key=os.getenv("GEMINI_API_KEY"),
    )
    result = llm.with_structured_output(Reflection).invoke(formatted_prompt)

    return {
        "is_sufficient": result.is_sufficient,
        "knowledge_gap": result.knowledge_gap,
        "follow_up_queries": result.follow_up_queries,
        "research_loop_count": state["research_loop_count"],
        "number_of_ran_queries": len(state["search_query"]),
    }
```

#### 关键点

| 元素 | 值 | 说明 |
|------|-----|------|
| 输入状态 | `OverallState` | 包含所有搜索结果 |
| 输出状态 | `ReflectionState` | 反思结果和后续查询 |
| 模型 | `gemini-2.5-flash` | 更强的推理能力 |
| Temperature | `1.0` | 鼓励探索性查询 |

#### 提示词分析

**reflection_instructions** (prompts.py:50-80):

```
- 识别知识缺口或需要深入探索的领域
- 如果摘要充分,不生成后续查询
- 如果有知识缺口,生成后续查询
- 关注技术细节、实现细节或未完全覆盖的新兴趋势
- 输出 JSON 格式
```

#### 输出示例

**研究不充分时**:
```json
{
  "is_sufficient": false,
  "knowledge_gap": "Missing information about cost trends and policy changes in renewable energy sector",
  "follow_up_queries": [
    "renewable energy cost trends 2024",
    "energy policy changes 2024"
  ]
}
```

**研究充分时**:
```json
{
  "is_sufficient": true,
  "knowledge_gap": "",
  "follow_up_queries": []
}
```

---

### 5. evaluate_research 路由函数

**文件位置**: `backend/src/agent/graph.py:183-217`

#### 功能
条件路由: 决定是继续研究还是生成最终答案

#### 实现细节

```python
def evaluate_research(state: ReflectionState, config: RunnableConfig) -> OverallState:
    configurable = Configuration.from_runnable_config(config)
    max_research_loops = (
        state.get("max_research_loops")
        if state.get("max_research_loops") is not None
        else configurable.max_research_loops
    )

    # 判断是否应该结束
    if state["is_sufficient"] or state["research_loop_count"] >= max_research_loops:
        return "finalize_answer"
    else:
        # 为每个后续查询创建并行任务
        return [
            Send(
                "web_research",
                {
                    "search_query": follow_up_query,
                    "id": state["number_of_ran_queries"] + int(idx),
                },
            )
            for idx, follow_up_query in enumerate(state["follow_up_queries"])
        ]
```

#### 决策逻辑

```
┌─────────────────────────────────────┐
│  is_sufficient == True?             │ → Yes → finalize_answer
└─────────────────────────────────────┘
           │ No
           ▼
┌─────────────────────────────────────┐
│  research_loop_count >= max_loops?  │ → Yes → finalize_answer
└─────────────────────────────────────┘
           │ No
           ▼
┌─────────────────────────────────────┐
│  继续研究 (生成并行 web_research)    │
└─────────────────────────────────────┘
```

#### 终止条件

1. **研究充分**: `is_sufficient == True`
2. **达到最大循环**: `research_loop_count >= max_research_loops`

#### 继续研究的逻辑

生成后续查询的并行任务:

```python
# 假设 follow_up_queries = ["cost trends", "policy changes"]
# number_of_ran_queries = 3 (之前的 3 个查询)

[
    Send("web_research", {"search_query": "cost trends", "id": 3}),
    Send("web_research", {"search_query": "policy changes", "id": 4})
]
```

---

### 6. finalize_answer 节点

**文件位置**: `backend/src/agent/graph.py:220-265`

#### 功能
基于所有收集的信息生成最终答案

#### 实现细节

```python
def finalize_answer(state: OverallState, config: RunnableConfig):
    configurable = Configuration.from_runnable_config(config)
    reasoning_model = state.get("reasoning_model") or configurable.answer_model

    # 格式化提示词
    formatted_prompt = answer_instructions.format(
        current_date=get_current_date(),
        research_topic=get_research_topic(state["messages"]),
        summaries="\n---\n\n".join(state["web_research_result"]),
    )

    # 初始化推理模型 (Gemini 2.5 Pro)
    llm = ChatGoogleGenerativeAI(
        model=reasoning_model,  # gemini-2.5-pro
        temperature=0,  # 低温度,确保准确性
        max_retries=2,
        api_key=os.getenv("GEMINI_API_KEY"),
    )
    result = llm.invoke(formatted_prompt)

    # 替换短 URL 为原始 URL
    unique_sources = []
    for source in state["sources_gathered"]:
        if source["short_url"] in result.content:
            result.content = result.content.replace(
                source["short_url"], source["value"]
            )
            unique_sources.append(source)

    return {
        "messages": [AIMessage(content=result.content)],
        "sources_gathered": unique_sources,
    }
```

#### 关键点

| 元素 | 值 | 说明 |
|------|-----|------|
| 输入状态 | `OverallState` | 所有搜索结果和来源 |
| 输出状态 | `OverallState` | 最终答案消息 |
| 模型 | `gemini-2.5-pro` | 最强的推理能力 |
| Temperature | `0` | 低温度,确保准确性 |

#### URL 替换逻辑

```python
# 摘要中的短 URL
"Renewable energy grew [1](https://vertexaisearch.cloud.google.com/id/0-0)"

# 替换为原始 URL
"Renewable energy grew [1](https://www.example.com/renewable-energy-2024)"
```

---

## 🔨 图构建流程

**文件位置**: `backend/src/agent/graph.py:268-293`

### 步骤 1: 创建 StateGraph

```python
builder = StateGraph(OverallState, config_schema=Configuration)
```

### 步骤 2: 添加节点

```python
builder.add_node("generate_query", generate_query)
builder.add_node("web_research", web_research)
builder.add_node("reflection", reflection)
builder.add_node("finalize_answer", finalize_answer)
```

### 步骤 3: 添加边

```python
# 入口点: START → generate_query
builder.add_edge(START, "generate_query")

# 条件边: generate_query → continue_to_web_research → web_research
builder.add_conditional_edges(
    "generate_query",
    continue_to_web_research,
    ["web_research"]
)

# 普通边: web_research → reflection
builder.add_edge("web_research", "reflection")

# 条件边: reflection → evaluate_research → {web_research, finalize_answer}
builder.add_conditional_edges(
    "reflection",
    evaluate_research,
    ["web_research", "finalize_answer"]
)

# 普通边: finalize_answer → END
builder.add_edge("finalize_answer", END)
```

### 步骤 4: 编译图

```python
graph = builder.compile(name="pro-search-agent")
```

---

## 🔄 完整执行流程示例

### 场景: 查询 "What are the latest trends in renewable energy?"

#### 初始状态

```python
{
    "messages": [HumanMessage("What are the latest trends in renewable energy?")],
    "search_query": [],
    "web_research_result": [],
    "sources_gathered": [],
    "initial_search_query_count": 3,
    "max_research_loops": 2,
    "research_loop_count": 0
}
```

#### 第 1 步: generate_query

**输入**: 用户问题
**输出**:
```python
{
    "search_query": [
        "renewable energy trends 2024",
        "solar energy advancements",
        "wind energy technology"
    ]
}
```

#### 第 2 步: continue_to_web_research

**输出**: 3 个并行任务
```python
[
    Send("web_research", {"search_query": "renewable energy trends 2024", "id": 0}),
    Send("web_research", {"search_query": "solar energy advancements", "id": 1}),
    Send("web_research", {"search_query": "wind energy technology", "id": 2})
]
```

#### 第 3 步: web_research (并行执行)

**输出** (累积):
```python
{
    "search_query": ["renewable energy trends 2024", "solar energy advancements", "wind energy technology"],
    "web_research_result": [
        "Summary of renewable energy trends... [1](short_url_0)",
        "Summary of solar advancements... [2](short_url_1)",
        "Summary of wind technology... [3](short_url_2)"
    ],
    "sources_gathered": [
        {"short_url": "short_url_0", "value": "https://...", "label": "..."},
        {"short_url": "short_url_1", "value": "https://...", "label": "..."},
        {"short_url": "short_url_2", "value": "https://...", "label": "..."}
    ]
}
```

#### 第 4 步: reflection

**输入**: 所有搜索结果
**输出**:
```python
{
    "is_sufficient": false,
    "knowledge_gap": "Missing cost and policy information",
    "follow_up_queries": ["renewable energy costs 2024", "energy policy 2024"],
    "research_loop_count": 1,
    "number_of_ran_queries": 3
}
```

#### 第 5 步: evaluate_research

**判断**: `is_sufficient == false` 且 `research_loop_count (1) < max_loops (2)`
**输出**: 继续研究
```python
[
    Send("web_research", {"search_query": "renewable energy costs 2024", "id": 3}),
    Send("web_research", {"search_query": "energy policy 2024", "id": 4})
]
```

#### 第 6 步: web_research (第二轮,并行)

**输出** (累积):
```python
{
    "web_research_result": [
        ... (之前的 3 个),
        "Summary of energy costs... [4](short_url_3)",
        "Summary of policy changes... [5](short_url_4)"
    ],
    "sources_gathered": [..., ... (新增 2 个)]
}
```

#### 第 7 步: reflection (第二轮)

**输出**:
```python
{
    "is_sufficient": true,
    "knowledge_gap": "",
    "follow_up_queries": [],
    "research_loop_count": 2,
    "number_of_ran_queries": 5
}
```

#### 第 8 步: evaluate_research

**判断**: `is_sufficient == true`
**输出**: `"finalize_answer"`

#### 第 9 步: finalize_answer

**输入**: 所有 5 个搜索结果和来源
**输出**:
```python
{
    "messages": [
        AIMessage("Based on my research, here are the latest trends in renewable energy... [1](original_url_0) ...")
    ],
    "sources_gathered": [...]
}
```

---

## 🎯 设计模式总结

### 1. 反思模式 (Reflection Pattern)

```
行动 → 反思 → 决策 → 行动 (迭代)
```

- **行动**: 执行搜索
- **反思**: 评估结果充分性
- **决策**: 继续或结束
- **迭代**: 最多 max_loops 次

### 2. 并行执行模式 (Parallel Execution)

```
一个查询列表 → 多个并行搜索任务 → 结果累积
```

- 使用 `Send` 对象创建并行分支
- 每个任务独立执行
- 结果通过 `operator.add` 自动累积

### 3. 条件路由模式 (Conditional Routing)

```
状态评估 → 条件判断 → 多个可能路径
```

- `evaluate_research` 根据状态决定下一步
- 可以返回字符串或 `Send` 对象列表
- 实现灵活的流程控制

---

## 💡 实践建议

### 1. 在 LangGraph UI 中可视化

访问 http://localhost:2024 查看:
- 图结构可视化
- 节点执行追踪
- 状态变化历史

### 2. 修改 max_research_loops

编辑 `backend/.env`:
```env
MAX_RESEARCH_LOOPS=5
```

观察迭代次数的变化。

### 3. 添加打印语句

```python
def generate_query(state: OverallState, config: RunnableConfig):
    print(f"[DEBUG] Generating queries for: {get_research_topic(state['messages'])}")
    # ... 原有代码
    print(f"[DEBUG] Generated queries: {result.query}")
    return {"search_query": result.query}
```

### 4. 添加新节点

```python
def my_custom_node(state: OverallState) -> dict:
    # 自定义逻辑
    return {"custom_field": "value"}

# 在图构建中添加
builder.add_node("my_custom_node", my_custom_node)
builder.add_edge("reflection", "my_custom_node")
builder.add_edge("my_custom_node", "evaluate_research")
```

---

## ✅ 阶段 2 总结

### 关键收获

1. **图结构**: LangGraph 使用 StateGraph 编排节点和边
2. **节点类型**: 普通节点、条件路由函数
3. **并行执行**: 使用 `Send` 对象创建并行任务
4. **条件路由**: 根据状态决定下一步
5. **反思模式**: 行动-反思-决策的迭代循环

### 核心概念图

```
┌──────────────┐
│  START       │
└──────┬───────┘
       ▼
┌──────────────┐       ┌──────────────┐
│ generate_    │──────→│ continue_to_ │
│    query     │       │ web_research │
└──────────────┘       └──────┬───────┘
                              ▼
                    ┌─────────────────────┐
                    │   web_research      │
                    │   (并行执行 xN)      │
                    └─────────┬───────────┘
                              ▼
                    ┌─────────────────────┐
                    │    reflection       │
                    └─────────┬───────────┘
                              ▼
                    ┌─────────────────────┐
                    │  evaluate_research  │
                    └───┬────────────┬────┘
                   不充分            充分
                        │              │
                        ▼              ▼
                ┌──────────────┐  ┌─────────────┐
                │ web_research │  │ finalize_   │
                │  (后续查询)   │  │   answer    │
                └──────────────┘  └──────┬──────┘
                                        │
                                        ▼
                                  ┌─────────┐
                                  │   END   │
                                  └─────────┘
```

### 下一步

进入**阶段 3: 提示词工程与工具使用**,深入学习:
- 提示词设计原则
- 引用处理机制
- 工具函数实现

### 学习验证

在进入下一阶段前,确保能够:

- [ ] 解释为什么需要 `continue_to_web_research`
- [ ] 理解并行搜索如何通过 `Send` 实现
- [ ] 能画出完整的状态转换流程图
- [ ] 理解每个节点的输入输出状态
- [ ] 能修改图结构添加新节点

---

## 📚 延伸阅读

- [LangGraph Graph Concepts](https://langchain-ai.github.io/langgraph/concepts/low_level/#graphs)
- [LangGraph Send Object](https://langchain-ai.github.io/langgraph/how_to/map_reduce/)
- [Google Grounding Metadata](https://ai.google.dev/gemini-api/docs/grounding)
- [Structured Output with LLMs](https://python.langchain.com/docs/how_to/structured_output/)

---

*下一阶段: 深入学习提示词工程与工具使用*
