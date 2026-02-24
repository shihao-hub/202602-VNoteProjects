# 阶段 3: 提示词工程与工具使用

## 📚 学习目标

- 理解系统提示词的设计原则
- 掌握结构化输出的提示技巧
- 学习引用处理和 URL 解析
- 深入理解工具函数的实现

---

## 🎯 提示词工程基础

### 什么是提示词工程?

提示词工程是设计和优化 LLM 输入,以获得高质量输出的技术。在本项目中:

- **结构化输出**: 使用 Pydantic BaseModel 强制输出格式
- **上下文注入**: 动态插入日期、主题等信息
- **示例引导**: 提供示例以指导模型输出
- **约束条件**: 明确指定要求和限制

---

## 📝 四个系统提示词详解

### 1. query_writer_instructions

**文件位置**: `backend/src/agent/prompts.py:9-34`

#### 目标
生成多样化的搜索查询

#### 完整提示词

```python
query_writer_instructions = """Your goal is to generate sophisticated and diverse web search queries. These queries are intended for an advanced automated web research tool capable of analyzing complex results, following links, and synthesizing information.

Instructions:
- Always prefer a single search query, only add another query if the original question requests multiple aspects or elements and one query is not enough.
- Each query should focus on one specific aspect of the original question.
- Don't produce more than {number_queries} queries.
- Queries should be diverse, if the topic is broad, generate more than 1 query.
- Don't generate multiple similar queries, 1 is enough.
- Query should ensure that the most current information is gathered. The current date is {current_date}.

Format:
- Format your response as a JSON object with ALL two of these exact keys:
   - "rationale": Brief explanation of why these queries are relevant
   - "query": A list of search queries

Example:

Topic: What revenue grew more last year apple stock or the number of people buying an iphone
```json
{{
    "rationale": "To answer this comparative growth question accurately, we need specific data points on Apple's stock performance and iPhone sales metrics. These queries target the precise financial information needed: company revenue trends, product-specific unit sales figures, and stock price movement over the same fiscal period for direct comparison.",
    "query": ["Apple total revenue growth fiscal year 2024", "iPhone unit sales growth fiscal year 2024", "Apple stock price growth fiscal year 2024"],
}}
```

Context: {research_topic}"""
```

#### 设计原则

| 原则 | 实现 | 效果 |
|------|------|------|
| **单一职责** | 每个查询关注一个特定方面 | 提高搜索精度 |
| **数量控制** | `Don't produce more than {number_queries}` | 避免过度搜索 |
| **多样性** | `Queries should be diverse` | 覆盖不同角度 |
| **去重** | `Don't generate multiple similar queries` | 避免重复搜索 |
| **时效性** | `The current date is {current_date}` | 获取最新信息 |
| **结构化输出** | JSON 格式示例 | 确保输出可解析 |

#### 技巧分析

**1. 变量占位符**
```python
{number_queries}     # 动态查询数量
{current_date}       # 当前日期
{research_topic}     # 研究主题
```

**2. 示例引导 (Few-Shot Prompting)**
```json
{
  "rationale": "...",
  "query": ["query1", "query2", "query3"]
}
```

提供具体示例,让模型理解期望的输出格式。

**3. 明确约束**
```python
- Don't produce more than {number_queries} queries.
- Don't generate multiple similar queries.
```

#### 使用方式

```python
formatted_prompt = query_writer_instructions.format(
    current_date=get_current_date(),  # "February 07, 2026"
    research_topic=get_research_topic(state["messages"]),
    number_queries=state["initial_search_query_count"],  # 3
)
```

#### 输出示例

**输入**: "What are the latest trends in renewable energy?"

**输出**:
```json
{
  "rationale": "These queries cover recent developments across different renewable energy sectors including overall trends, solar technology, and wind energy advancements.",
  "query": [
    "renewable energy trends 2024",
    "solar energy technology advancements 2024",
    "wind energy innovations 2024"
  ]
}
```

---

### 2. web_searcher_instructions

**文件位置**: `backend/src/agent/prompts.py:37-48`

#### 目标
执行搜索并总结结果

#### 完整提示词

```python
web_searcher_instructions = """Conduct targeted Google Searches to gather the most recent, credible information on "{research_topic}" and synthesize it into a verifiable text artifact.

Instructions:
- Query should ensure that the most current information is gathered. The current date is {current_date}.
- Conduct multiple, diverse searches to gather comprehensive information.
- Consolidate key findings while meticulously tracking the source(s) for each specific piece of information.
- The output should be a well-written summary or report based on your search findings.
- Only include the information found in the search results, don't make up any information.

Research Topic:
{research_topic}
"""
```

#### 设计原则

| 原则 | 实现 | 效果 |
|------|------|------|
| **时效性** | `The current date is {current_date}` | 优先搜索最新内容 |
| **综合性** | `Conduct multiple, diverse searches` | 全面收集信息 |
| **可验证性** | `tracking the source(s)` | 确保信息可追溯 |
| **真实性** | `Only include the information found in the search results` | 避免幻觉 |
| **禁止编造** | `don't make up any information` | 确保准确性 |

#### 关键点

这个提示词配合 Google Search API 使用:

```python
response = genai_client.models.generate_content(
    model="gemini-2.0-flash",
    contents=formatted_prompt,
    config={
        "tools": [{"google_search": {}}],  # 启用搜索
        "temperature": 0,
    },
)
```

`grounding_metadata` 包含:
- 搜索结果的来源 URL
- 引用的位置索引
- 用于生成带引用的答案

#### 输出示例

```markdown
Renewable energy capacity reached new highs in 2024, with solar and wind leading the growth. Global renewable capacity increased by 25% compared to 2023, driven by declining costs and supportive policies.
```

后续会通过 `insert_citation_markers` 插入引用标记。

---

### 3. reflection_instructions

**文件位置**: `backend/src/agent/prompts.py:50-80`

#### 目标
识别知识缺口,生成后续查询

#### 完整提示词

```python
reflection_instructions = """You are an expert research assistant analyzing summaries about "{research_topic}".

Instructions:
- Identify knowledge gaps or areas that need deeper exploration and generate a follow-up query. (1 or multiple).
- If provided summaries are sufficient to answer the user's question, don't generate a follow-up query.
- If there is a knowledge gap, generate a follow-up query that would help expand your understanding.
- Focus on technical details, implementation specifics, or emerging trends that weren't fully covered.

Requirements:
- Ensure the follow-up query is self-contained and includes necessary context for web search.

Output Format:
- Format your response as a JSON object with these exact keys:
   - "is_sufficient": true or false
   - "knowledge_gap": Describe what information is missing or needs clarification
   - "follow_up_queries": Write a specific question to address this gap

Example:
```json
{{
    "is_sufficient": true, // or false
    "knowledge_gap": "The summary lacks information about performance metrics and benchmarks", // "" if is_sufficient is true
    "follow_up_queries": ["What are typical performance benchmarks and metrics used to evaluate [specific technology]?"] // [] if is_sufficient is true
}}
```

Reflect carefully on the Summaries to identify knowledge gaps and produce a follow-up query. Then, produce your output following this JSON format:

Summaries:
{summaries}
"""
```

#### 设计原则

| 原则 | 实现 | 效果 |
|------|------|------|
| **自我评估** | `is_sufficient` | 判断研究是否充分 |
| **缺口识别** | `knowledge_gap` | 明确缺失的信息 |
| **后续行动** | `follow_up_queries` | 生成改进查询 |
| **自包含** | `Ensure the follow-up query is self-contained` | 查询独立可执行 |
| **聚焦细节** | `Focus on technical details, implementation specifics` | 深入挖掘 |

#### 关键点

**双重输出模式**:

1. **研究充分时**:
```json
{
  "is_sufficient": true,
  "knowledge_gap": "",
  "follow_up_queries": []
}
```

2. **研究不充分时**:
```json
{
  "is_sufficient": false,
  "knowledge_gap": "Missing information about cost trends and policy changes",
  "follow_up_queries": [
    "renewable energy cost trends 2024",
    "energy policy changes 2024"
  ]
}
```

#### 使用方式

```python
formatted_prompt = reflection_instructions.format(
    current_date=get_current_date(),
    research_topic=get_research_topic(state["messages"]),
    summaries="\n\n---\n\n".join(state["web_research_result"]),
)
```

将所有搜索结果用 `\n\n---\n\n` 分隔,形成清晰的总结。

---

### 4. answer_instructions

**文件位置**: `backend/src/agent/prompts.py:82-96`

#### 目标
生成带引用的最终答案

#### 完整提示词

```python
answer_instructions = """Generate a high-quality answer to the user's question based on the provided summaries.

Instructions:
- The current date is {current_date}.
- You are the final step of a multi-step research process, don't mention that you are the final step.
- You have access to all the information gathered from the previous steps.
- You have access to the user's question.
- Generate a high-quality answer to the user's question based on the provided summaries and the user's question.
- Include the sources you used from the Summaries in the answer correctly, use markdown format (e.g. [apnews](https://vertexaisearch.cloud.google.com/id/1-0)). THIS IS A MUST.

User Context:
- {research_topic}

Summaries:
{summaries}"""
```

#### 设计原则

| 原则 | 实现 | 效果 |
|------|------|------|
| **自然表达** | `don't mention that you are the final step` | 提升用户体验 |
| **完整性** | `You have access to all the information` | 整合所有研究结果 |
| **引用必须** | `THIS IS A MUST` | 确保引用正确 |
| **Markdown 格式** | `[text](url)` | 可读的链接格式 |

#### 关键点

**引用格式要求**:
```markdown
[标题或描述](https://vertexaisearch.cloud.google.com/id/{search_id}-{chunk_index})
```

示例:
```markdown
Renewable energy capacity grew by 25% in 2024 [IEA Report](https://vertexaisearch.cloud.google.com/id/0-0).
```

#### URL 替换流程

1. **初始生成**: 使用短 URL
2. **finalize_answer**: 替换为原始 URL
3. **最终输出**: 带完整引用的答案

```python
# 在 finalize_answer 节点中
for source in state["sources_gathered"]:
    if source["short_url"] in result.content:
        result.content = result.content.replace(
            source["short_url"], source["value"]  # 替换为原始 URL
        )
```

---

## 🛠️ 工具函数深度解析

### 1. get_research_topic

**文件位置**: `backend/src/agent/utils.py:5-19`

#### 功能
从消息历史中提取研究主题

#### 实现

```python
def get_research_topic(messages: List[AnyMessage]) -> str:
    """
    Get the research topic from the messages.
    """
    # 单轮对话
    if len(messages) == 1:
        research_topic = messages[-1].content
    # 多轮对话
    else:
        research_topic = ""
        for message in messages:
            if isinstance(message, HumanMessage):
                research_topic += f"User: {message.content}\n"
            elif isinstance(message, AIMessage):
                research_topic += f"Assistant: {message.content}\n"
    return research_topic
```

#### 使用场景

| 场景 | 输入 | 输出 |
|------|------|------|
| 单轮对话 | `[HumanMessage("What is AI?")]` | `"What is AI?"` |
| 多轮对话 | `[HumanMessage("Hi"), AIMessage("Hello"), HumanMessage("What is AI?")]` | `"User: Hi\nAssistant: Hello\nUser: What is AI?\n"` |

#### 关键点

- **单轮对话**: 直接返回最后一条消息
- **多轮对话**: 拼接完整对话历史,提供上下文

---

### 2. resolve_urls

**文件位置**: `backend/src/agent/utils.py:22-36`

#### 功能
将长 URL 转换为短 URL

#### 实现

```python
def resolve_urls(urls_to_resolve: List[Any], id: int) -> Dict[str, str]:
    """
    Create a map of the vertex ai search urls (very long) to a short url with a unique id for each url.
    Ensures each original URL gets a consistent shortened form while maintaining uniqueness.
    """
    prefix = f"https://vertexaisearch.cloud.google.com/id/"
    urls = [site.web.uri for site in urls_to_resolve]

    # 创建映射表
    resolved_map = {}
    for idx, url in enumerate(urls):
        if url not in resolved_map:
            resolved_map[url] = f"{prefix}{id}-{idx}"

    return resolved_map
```

#### 为什么需要短 URL?

1. **节省 Token**: 长 URL 可能占用数百字符
2. **提升性能**: 减少传输和处理时间
3. **可读性**: 短 URL 更易读
4. **一致性**: 同一个 URL 使用相同的短形式

#### 示例

**输入**:
```python
urls_to_resolve = [
    GroundingChunk(web.uri="https://www.example.com/very/long/url/1"),
    GroundingChunk(web.uri="https://www.example.com/very/long/url/2")
]
id = 0
```

**输出**:
```python
{
    "https://www.example.com/very/long/url/1": "https://vertexaisearch.cloud.google.com/id/0-0",
    "https://www.example.com/very/long/url/2": "https://vertexaisearch.cloud.google.com/id/0-1"
}
```

#### URL 格式

```
https://vertexaisearch.cloud.google.com/id/{search_id}-{chunk_index}
                                    ↑              ↑
                                搜索任务 ID      引用索引
```

---

### 3. get_citations

**文件位置**: `backend/src/agent/utils.py:78-166`

#### 功能
从 Gemini 的 grounding_metadata 提取引用

#### 实现分析

```python
def get_citations(response, resolved_urls_map):
    """
    Extracts and formats citation information from a Gemini model's response.
    """
    citations = []

    # 确保 response 和必要结构存在
    if not response or not response.candidates:
        return citations

    candidate = response.candidates[0]
    if (
        not hasattr(candidate, "grounding_metadata")
        or not candidate.grounding_metadata
        or not hasattr(candidate.grounding_metadata, "grounding_supports")
    ):
        return citations

    # 遍历所有 grounding_supports
    for support in candidate.grounding_metadata.grounding_supports:
        citation = {}

        # 确保 segment 信息存在
        if not hasattr(support, "segment") or support.segment is None:
            continue

        # 获取索引
        start_index = support.segment.start_index if support.segment.start_index is not None else 0

        if support.segment.end_index is None:
            continue

        citation["start_index"] = start_index
        citation["end_index"] = support.segment.end_index
        citation["segments"] = []

        # 处理 grounding_chunk_indices
        if (
            hasattr(support, "grounding_chunk_indices")
            and support.grounding_chunk_indices
        ):
            for ind in support.grounding_chunk_indices:
                try:
                    chunk = candidate.grounding_metadata.grounding_chunks[ind]
                    resolved_url = resolved_urls_map.get(chunk.web.uri, None)
                    citation["segments"].append(
                        {
                            "label": chunk.web.title.split(".")[:-1][0],
                            "short_url": resolved_url,
                            "value": chunk.web.uri,
                        }
                    )
                except (IndexError, AttributeError, NameError):
                    pass

        citations.append(citation)

    return citations
```

#### grounding_metadata 结构

```
response.candidates[0].grounding_metadata
    ├── grounding_chunks (搜索结果列表)
    │   └── [0]
    │       └── web
    │           ├── uri (原始 URL)
    │           └── title (标题)
    └── grounding_supports (引用位置列表)
        └── [0]
            └── segment
                ├── start_index (开始位置)
                └── end_index (结束位置)
            └── grounding_chunk_indices (引用的 chunk 索引)
                └── [0, 1]  # 引用了 chunk[0] 和 chunk[1]
```

#### 输出示例

```python
[
    {
        "start_index": 50,
        "end_index": 100,
        "segments": [
            {
                "label": "IEA Report 2024",
                "short_url": "https://vertexaisearch.cloud.google.com/id/0-0",
                "value": "https://www.iea.org/reports/2024"
            }
        ]
    }
]
```

---

### 4. insert_citation_markers

**文件位置**: `backend/src/agent/utils.py:39-75`

#### 功能
在文本中插入引用标记

#### 实现

```python
def insert_citation_markers(text, citations_list):
    """
    Inserts citation markers into a text string based on start and end indices.
    """
    # 从后向前排序,避免索引偏移
    sorted_citations = sorted(
        citations_list,
        key=lambda c: (c["end_index"], c["start_index"]),
        reverse=True
    )

    modified_text = text
    for citation_info in sorted_citations:
        end_idx = citation_info["end_index"]
        marker_to_insert = ""
        for segment in citation_info["segments"]:
            marker_to_insert += f" [{segment['label']}]({segment['short_url']})"

        # 在 end_idx 位置插入引用标记
        modified_text = (
            modified_text[:end_idx] + marker_to_insert + modified_text[end_idx:]
        )

    return modified_text
```

#### 关键技巧: 从后向前插入

**为什么从后向前?**

假设文本: `"ABCDEF"`

**从前向后插入** (错误):
```
位置 2 插入 "X" → "ABCXDEF"
位置 4 插入 "Y" → "ABCXDYEF"  ❌ 位置偏移了!
```

**从后向前插入** (正确):
```
位置 4 插入 "Y" → "ABCDEFY"
位置 2 插入 "X" → "ABCXDYEF"  ✅ 正确!
```

#### 示例

**输入**:
```python
text = "Renewable energy grew 25% in 2024"
citations = [
    {
        "start_index": 0,
        "end_index": 30,
        "segments": [
            {"label": "IEA", "short_url": "https://vertexaisearch.cloud.google.com/id/0-0"}
        ]
    }
]
```

**输出**:
```markdown
Renewable energy grew 25% in 2024 [IEA](https://vertexaisearch.cloud.google.com/id/0-0)
```

---

## 🎨 引用处理完整流程

### 流程图

```
Google Search API
    ↓
grounding_metadata
    ├── grounding_chunks (来源信息)
    └─ grounding_supports (引用位置)
    ↓
resolve_urls (转换为短 URL)
    ├── https://example.com/long/url/1 → short_url_0
    └── https://example.com/long/url/2 → short_url_1
    ↓
get_citations (提取引用对象)
    └── [
          {
            "start_index": 50,
            "end_index": 100,
            "segments": [
              {"label": "Source 1", "short_url": "short_url_0"}
            ]
          }
        ]
    ↓
insert_citation_markers (插入引用标记)
    └── "Text... [Source 1](short_url_0)"
    ↓
finalize_answer (替换为原始 URL)
    └── "Text... [Source 1](https://example.com/long/url/1)"
```

---

## 💡 提示词设计最佳实践

### 1. 结构化输出技巧

**使用 Pydantic BaseModel**:
```python
class SearchQueryList(BaseModel):
    query: List[str] = Field(description="...")
    rationale: str = Field(description="...")
```

**在提示词中明确格式**:
```python
"""
Format your response as a JSON object with these exact keys:
   - "rationale": ...
   - "query": ...
"""
```

**使用 `with_structured_output`**:
```python
structured_llm = llm.with_structured_output(SearchQueryList)
result = structured_llm.invoke(prompt)
```

### 2. 变量注入模式

```python
# 使用 .format() 注入变量
prompt = template.format(
    current_date=get_current_date(),
    research_topic=topic,
    number_queries=3,
)
```

### 3. 示例引导 (Few-Shot Prompting)

```python
"""
Example:

Topic: What is AI?
```json
{{
    "rationale": "...",
    "query": ["query1", "query2"]
}}
```
"""
```

### 4. 约束明确化

```python
"""
- Don't produce more than {number_queries} queries.
- Only include the information found in the search results.
- THIS IS A MUST.
"""
```

---

## ✅ 阶段 3 总结

### 关键收获

1. **提示词设计**: 四个系统提示词各有明确目标
2. **结构化输出**: 使用 Pydantic BaseModel 确保格式正确
3. **引用处理**: 完整的 URL 解析和引用插入流程
4. **工具函数**: resolve_urls, get_citations, insert_citation_markers
5. **从后向前插入**: 避免索引偏移的关键技巧

### 核心概念图

```
提示词工程
    ├── query_writer_instructions (查询生成)
    ├── web_searcher_instructions (搜索总结)
    ├── reflection_instructions (反思分析)
    └── answer_instructions (最终答案)

工具函数
    ├── get_research_topic (提取主题)
    ├── resolve_urls (URL 缩短)
    ├── get_citations (提取引用)
    └── insert_citation_markers (插入引用)

引用处理流程
    Google Search → grounding_metadata
        → resolve_urls → get_citations
        → insert_citation_markers → 最终答案
```

### 下一步

进入**阶段 4: 前端架构与流式通信**,深入学习:
- React 组件架构
- useStream Hook
- 实时活动时间线

### 学习验证

在进入下一阶段前,确保能够:

- [ ] 理解每个提示词的设计意图
- [ ] 能解释为什么 citations 需要从后向前插入
- [ ] 理解 grounding_metadata 的作用
- [ ] 能设计自己的提示词模板
- [ ] 理解引用处理的完整流程

---

## 📚 延伸阅读

- [Prompt Engineering Guide](https://www.promptingguide.ai/)
- [Few-Shot Prompting](https://www.promptingguide.ai/techniques/fewshot)
- [Structured Output](https://python.langchain.com/docs/how_to/structured_output/)
- [Google Grounding Docs](https://ai.google.dev/gemini-api/docs/grounding)

---

*下一阶段: 深入学习前端架构与流式通信*
