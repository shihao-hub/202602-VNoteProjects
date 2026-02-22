# 阶段 4: 前端架构与流式通信

## 📚 学习目标

- 理解 React 组件架构
- 掌握 LangGraph SDK 的 useStream Hook
- 理解实时活动时间线的实现
- 学习 Markdown 渲染和自定义组件
- 掌握前后端流式通信机制

---

## 🎯 前端技术栈

### 核心框架
- **React 18** - 用户界面框架
- **TypeScript** - 类型安全
- **Vite** - 构建工具

### UI 组件库
- **Tailwind CSS** - 样式框架
- **shadcn/ui** - UI 组件库
- **Lucide React** - 图标库

### LangGraph 集成
- **@langchain/langgraph-sdk/react** - LangGraph React SDK

### Markdown 渲染
- **react-markdown** - Markdown 解析和渲染

---

## 📊 组件层次结构

```
App (主应用)
    ├── WelcomeScreen (欢迎屏幕)
    └── ChatMessagesView (聊天消息视图)
        ├── ActivityTimeline (活动时间线)
        ├── HumanMessageBubble (用户消息气泡)
        ├── AiMessageBubble (AI 消息气泡)
        └── InputForm (输入表单)
```

---

## 🔑 useStream Hook 核心解析

**文件位置**: `frontend/src/App.tsx:19-72`

### 基本用法

```typescript
const thread = useStream<{
  messages: Message[];
  initial_search_query_count: number;
  max_research_loops: number;
  reasoning_model: string;
}>({
  apiUrl: import.meta.env.DEV
    ? "http://localhost:2024"   // 开发环境
    : "http://localhost:8123",  // 生产环境
  assistantId: "agent",
  messagesKey: "messages",
  onUpdateEvent: (event: any) => { /* 处理事件 */ },
  onError: (error: any) => { /* 处理错误 */ },
});
```

### 参数说明

| 参数 | 值 | 说明 |
|------|-----|------|
| `apiUrl` | 开发: `localhost:2024`<br>生产: `localhost:8123` | LangGraph API 地址 |
| `assistantId` | `"agent"` | 助手 ID,对应 langgraph.json 中的图名称 |
| `messagesKey` | `"messages"` | 状态中消息的键名 |
| `onUpdateEvent` | 回调函数 | 处理流式更新事件 |
| `onError` | 回调函数 | 处理错误 |

### thread 对象属性

```typescript
thread.messages      // 消息历史
thread.isLoading     // 是否正在加载
thread.submit()      // 提交新消息
thread.stop()        // 停止当前任务
```

---

## 📡 事件流处理

### 事件类型

**文件位置**: `frontend/src/App.tsx:30-67`

```typescript
onUpdateEvent: (event: any) => {
  let processedEvent: ProcessedEvent | null = null;

  // 1. 生成查询事件
  if (event.generate_query) {
    processedEvent = {
      title: "Generating Search Queries",
      data: event.generate_query?.search_query?.join(", ") || "",
    };
  }

  // 2. 网络研究事件
  else if (event.web_research) {
    const sources = event.web_research.sources_gathered || [];
    const numSources = sources.length;
    const uniqueLabels = [...new Set(sources.map((s: any) => s.label).filter(Boolean))];
    const exampleLabels = uniqueLabels.slice(0, 3).join(", ");
    processedEvent = {
      title: "Web Research",
      data: `Gathered ${numSources} sources. Related to: ${exampleLabels || "N/A"}.`,
    };
  }

  // 3. 反思事件
  else if (event.reflection) {
    processedEvent = {
      title: "Reflection",
      data: "Analysing Web Research Results",
    };
  }

  // 4. 最终答案事件
  else if (event.finalize_answer) {
    processedEvent = {
      title: "Finalizing Answer",
      data: "Composing and presenting the final answer.",
    };
    hasFinalizeEventOccurredRef.current = true;
  }

  if (processedEvent) {
    setProcessedEventsTimeline((prevEvents) => [...prevEvents, processedEvent!]);
  }
},
```

### 事件与后端节点的映射

| 前端事件 | 后端节点 | 数据示例 |
|---------|---------|---------|
| `generate_query` | `generate_query` | 搜索查询列表 |
| `web_research` | `web_research` | 来源信息、引用 |
| `reflection` | `reflection` | 分析状态 |
| `finalize_answer` | `finalize_answer` | 最终答案 |

---

## 🔄 状态管理

### 1. processedEventsTimeline (实时活动)

**文件位置**: `frontend/src/App.tsx:10-12`

```typescript
const [processedEventsTimeline, setProcessedEventsTimeline] = useState<ProcessedEvent[]>([]);
```

**用途**: 存储当前正在执行的活动事件

**生命周期**:
- 新任务开始时清空
- 随着事件流更新累积
- 任务完成后转移到 historicalActivities

### 2. historicalActivities (历史活动)

**文件位置**: `frontend/src/App.tsx:13-15`

```typescript
const [historicalActivities, setHistoricalActivities] = useState<Record<string, ProcessedEvent[]>>({});
```

**用途**: 缓存已完成任务的活动时间线

**数据结构**:
```typescript
{
  "message-id-1": [ProcessedEvent, ProcessedEvent, ...],
  "message-id-2": [ProcessedEvent, ProcessedEvent, ...],
  // ...
}
```

### 3. 活动转移逻辑

**文件位置**: `frontend/src/App.tsx:85-100`

```typescript
useEffect(() => {
  if (
    hasFinalizeEventOccurredRef.current &&  // 已完成
    !thread.isLoading &&                    // 不在加载
    thread.messages.length > 0              // 有消息
  ) {
    const lastMessage = thread.messages[thread.messages.length - 1];
    if (lastMessage && lastMessage.type === "ai" && lastMessage.id) {
      setHistoricalActivities((prev) => ({
        ...prev,
        [lastMessage.id!]: [...processedEventsTimeline],  // 保存到历史
      }));
    }
    hasFinalizeEventOccurredRef.current = false;
  }
}, [thread.messages, thread.isLoading, processedEventsTimeline]);
```

---

## 🎨 活动时间线组件

### ActivityTimeline 组件

**文件位置**: `frontend/src/components/ActivityTimeline.tsx`

### Props

```typescript
interface ActivityTimelineProps {
  processedEvents: ProcessedEvent[];  // 活动事件列表
  isLoading: boolean;                  // 是否正在加载
}
```

### 图标映射

**文件位置**: `frontend/src/components/ActivityTimeline.tsx:37-53`

```typescript
const getEventIcon = (title: string, index: number) => {
  if (title.toLowerCase().includes("generating")) {
    return <TextSearch className="h-4 w-4 text-neutral-400" />;
  } else if (title.toLowerCase().includes("reflection")) {
    return <Brain className="h-4 w-4 text-neutral-400" />;
  } else if (title.toLowerCase().includes("research")) {
    return <Search className="h-4 w-4 text-neutral-400" />;
  } else if (title.toLowerCase().includes("finalizing")) {
    return <Pen className="h-4 w-4 text-neutral-400" />;
  }
  return <Activity className="h-4 w-4 text-neutral-400" />;
};
```

### 可折叠功能

```typescript
const [isTimelineCollapsed, setIsTimelineCollapsed] = useState<boolean>(false);

// 自动折叠: 完成后自动折叠
useEffect(() => {
  if (!isLoading && processedEvents.length !== 0) {
    setIsTimelineCollapsed(true);
  }
}, [isLoading, processedEvents]);
```

### 时间线渲染

**结构**:
```
┌─ Research (可折叠) ─────────────────┐
│ ○ Generating Search Queries         │
│ │  query1, query2, query3           │
│ │                                   │
│ ○ Web Research                      │
│ │  Gathered 5 sources. Related to...│
│ │                                   │
│ ○ Reflection                        │
│ │  Analysing Web Research Results   │
│ │                                   │
│ ○ Finalizing Answer                 │
│    Composing final answer...        │
└─────────────────────────────────────┘
```

---

## 💬 消息渲染

### HumanMessageBubble (用户消息)

**文件位置**: `frontend/src/components/ChatMessagesView.tsx:144-158`

```typescript
const HumanMessageBubble: React.FC<HumanMessageBubbleProps> = ({
  message,
  mdComponents,
}) => {
  return (
    <div className="text-white rounded-3xl break-words min-h-7 bg-neutral-700 max-w-[100%] sm:max-w-[90%] px-4 pt-3 rounded-br-lg">
      <ReactMarkdown components={mdComponents}>
        {typeof message.content === "string"
          ? message.content
          : JSON.stringify(message.content)}
      </ReactMarkdown>
    </div>
  );
};
```

**样式特点**:
- 右对齐
- 灰色背景 (`bg-neutral-700`)
- 圆角矩形
- 最大宽度 90%

### AiMessageBubble (AI 消息)

**文件位置**: `frontend/src/components/ChatMessagesView.tsx:174-223`

```typescript
const AiMessageBubble: React.FC<AiMessageBubbleProps> = ({
  message,
  historicalActivity,
  liveActivity,
  isLastMessage,
  isOverallLoading,
  mdComponents,
  handleCopy,
  copiedMessageId,
}) => {
  // 决定显示哪个活动时间线
  const activityForThisBubble =
    isLastMessage && isOverallLoading ? liveActivity : historicalActivity;

  return (
    <div className="relative break-words flex flex-col">
      {/* 活动时间线 */}
      {activityForThisBubble && activityForThisBubble.length > 0 && (
        <div className="mb-3 border-b border-neutral-700 pb-3 text-xs">
          <ActivityTimeline
            processedEvents={activityForThisBubble}
            isLoading={isLiveActivityForThisBubble}
          />
        </div>
      )}

      {/* Markdown 内容 */}
      <ReactMarkdown components={mdComponents}>
        {typeof message.content === "string"
          ? message.content
          : JSON.stringify(message.content)}
      </ReactMarkdown>

      {/* 复制按钮 */}
      <Button onClick={() => handleCopy(message.content, message.id!)}>
        {copiedMessageId === message.id ? "Copied" : "Copy"}
      </Button>
    </div>
  );
};
```

**样式特点**:
- 左对齐
- 包含活动时间线
- Markdown 渲染
- 复制按钮

---

## 📝 Markdown 自定义组件

### mdComponents 对象

**文件位置**: `frontend/src/components/ChatMessagesView.tsx:24-135`

```typescript
const mdComponents = {
  h1: ({ className, children, ...props }) => (
    <h1 className="text-2xl font-bold mt-4 mb-2" {...props}>
      {children}
    </h1>
  ),
  p: ({ className, children, ...props }) => (
    <p className="mb-3 leading-7" {...props}>
      {children}
    </p>
  ),
  a: ({ className, children, href, ...props }) => (
    <Badge className="text-xs mx-0.5">
      <a
        className="text-blue-400 hover:text-blue-300 text-xs"
        href={href}
        target="_blank"
        rel="noopener noreferrer"
        {...props}
      >
        {children}
      </a>
    </Badge>
  ),
  // ... 更多组件
};
```

### 特殊处理: 链接转换为 Badge

```typescript
a: ({ className, children, href, ...props }) => (
  <Badge className="text-xs mx-0.5">  {/* 使用 Badge 组件 */}
    <a
      className="text-blue-400 hover:text-blue-300 text-xs"
      href={href}
      target="_blank"
      rel="noopener noreferrer"
    >
      {children}
    </a>
  </Badge>
),
```

**效果**: 将引用链接显示为可点击的 Badge 样式

---

## 🚀 提交流程

### handleSubmit 函数

**文件位置**: `frontend/src/App.tsx:102-145`

```typescript
const handleSubmit = useCallback(
  (submittedInputValue: string, effort: string, model: string) => {
    if (!submittedInputValue.trim()) return;

    // 清空当前活动时间线
    setProcessedEventsTimeline([]);
    hasFinalizeEventOccurredRef.current = false;

    // 将 effort 转换为参数
    let initial_search_query_count = 0;
    let max_research_loops = 0;
    switch (effort) {
      case "low":
        initial_search_query_count = 1;
        max_research_loops = 1;
        break;
      case "medium":
        initial_search_query_count = 3;
        max_research_loops = 3;
        break;
      case "high":
        initial_search_query_count = 5;
        max_research_loops = 10;
        break;
    }

    // 构建新消息
    const newMessages: Message[] = [
      ...(thread.messages || []),
      {
        type: "human",
        content: submittedInputValue,
        id: Date.now().toString(),
      },
    ];

    // 提交到后端
    thread.submit({
      messages: newMessages,
      initial_search_query_count,
      max_research_loops,
      reasoning_model: model,
    });
  },
  [thread]
);
```

### effort 参数映射

| Effort | 查询数量 | 最大循环 | 适用场景 |
|--------|---------|---------|---------|
| `low` | 1 | 1 | 简单问题,快速回答 |
| `medium` | 3 | 3 | 中等复杂度,平衡速度和质量 |
| `high` | 5 | 10 | 复杂研究,追求深度和全面性 |

---

## 🌐 前后端通信流程

### 流式通信架构

```
前端 (React)
    │
    ├─ useStream Hook
    │   │
    │   ├─ WebSocket 连接到 LangGraph API
    │   │
    │   ├─ thread.submit() 提交消息
    │   │   ↓
    │   │   POST /threads/{thread_id}/runs
    │   │   Body: {
    │   │     messages: [...],
    │   │     initial_search_query_count: 3,
    │   │     max_research_loops: 2
    │   │   }
    │   │
    │   └─ onUpdateEvent() 接收事件
    │       ↑
    │       SSE (Server-Sent Events)
    │       数据流:
    │       - generate_query 事件
    │       - web_research 事件
    │       - reflection 事件
    │       - finalize_answer 事件
    │
后端 (LangGraph API)
    │
    ├─ FastAPI + LangGraph
    │   └─ /runs endpoint
    │
    └─ StateGraph 执行
        ├─ generate_query 节点
        ├─ web_research 节点
        ├─ reflection 节点
        └─ finalize_answer 节点
```

### SSE 事件格式

```typescript
// 前端接收的事件格式
{
  generate_query: {
    search_query: ["query1", "query2", "query3"]
  }
}

{
  web_research: {
    sources_gathered: [
      { label: "Source 1", short_url: "...", value: "..." }
    ]
  }
}

{
  reflection: {
    is_sufficient: false,
    knowledge_gap: "Missing information...",
    follow_up_queries: ["query4", "query5"]
  }
}

{
  finalize_answer: {
    messages: [{ type: "ai", content: "Final answer..." }]
  }
}
```

---

## 💡 实践建议

### 1. 修改 effort 参数映射

```typescript
// 自定义映射
switch (effort) {
  case "quick":
    initial_search_query_count = 1;
    max_research_loops = 0;  // 不迭代
    break;
  case "thorough":
    initial_search_query_count = 10;
    max_research_loops = 20;  // 深度研究
    break;
}
```

### 2. 添加新的事件类型

```typescript
// 在 onUpdateEvent 中添加
else if (event.my_custom_node) {
  processedEvent = {
    title: "My Custom Node",
    data: event.my_custom_node.custom_data,
  };
}

// 在 getEventIcon 中添加图标
if (title.toLowerCase().includes("custom")) {
  return <CustomIcon className="h-4 w-4" />;
}
```

### 3. 实现消息持久化

```typescript
// 使用 localStorage
useEffect(() => {
  localStorage.setItem("chat-history", JSON.stringify(thread.messages));
}, [thread.messages]);

// 加载历史
const loadHistory = () => {
  const saved = localStorage.getItem("chat-history");
  return saved ? JSON.parse(saved) : [];
};
```

### 4. 添加新的配置选项

```typescript
// 在 InputForm 中添加新选项
<select value={temperature} onChange={(e) => setTemperature(e.target.value)}>
  <option value="0">Conservative (0.0)</option>
  <option value="0.7">Balanced (0.7)</option>
  <option value="1">Creative (1.0)</option>
</select>

// 在 handleSubmit 中传递
thread.submit({
  messages: newMessages,
  temperature: parseFloat(temperature),
});
```

---

## ✅ 阶段 4 总结

### 关键收获

1. **useStream Hook**: LangGraph SDK 的核心,处理流式通信
2. **事件流处理**: onUpdateEvent 将后端事件转换为 UI 更新
3. **状态管理**: processedEventsTimeline (实时) + historicalActivities (历史)
4. **活动时间线**: 可视化 Agent 的执行过程
5. **Markdown 渲染**: 自定义组件实现样式化输出

### 核心概念图

```
前端架构
    ├── App (主应用)
    │   ├── useStream (流式通信)
    │   ├── processedEventsTimeline (实时活动)
    │   └── historicalActivities (历史活动)
    │
    ├── ChatMessagesView (聊天视图)
    │   ├── HumanMessageBubble (用户消息)
    │   ├── AiMessageBubble (AI 消息)
    │   │   └── ActivityTimeline (活动时间线)
    │   └── InputForm (输入表单)
    │
    └── Markdown 组件
        ├── h1, h2, h3, p
        ├── a → Badge (引用链接)
        ├── ul, ol, li
        └── code, pre
```

### 下一步

进入**阶段 5: 前后端通信与部署**,深入学习:
- FastAPI 集成
- LangGraph API 配置
- Docker 部署

### 学习验证

在进入下一阶段前,确保能够:

- [ ] 理解 useStream 的工作原理
- [ ] 能解释为什么需要 historicalActivities
- [ ] 理解事件流如何转换为 UI 更新
- [ ] 能修改 effort 到参数的映射逻辑
- [ ] 理解前后端流式通信的机制

---

## 📚 延伸阅读

- [LangGraph SDK React Documentation](https://langchain-ai.github.io/langgraph/concepts/react_streaming/)
- [React Hooks](https://react.dev/reference/react)
- [React Markdown](https://github.com/remarkjs/react-markdown)
- [Server-Sent Events (SSE)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events)

---

*下一阶段: 深入学习前后端通信与部署*
