ReAct（Reasoning + Acting）是一种结合**推理（Reasoning）与行动（Acting）的智能体（Agent）交互范式，旨在提升大语言模型（LLMs）在复杂任务中的决策能力、可解释性与任务完成效率。该模式由 Yao et al. 在 2022 年提出（论文：*ReAct: Synergizing Reasoning and Acting in Language Models*），其核心思想是让模型在执行任务时交替进行思考（Thought）和行动（Action）**，并通过与外部环境（如工具、API、数据库或模拟器）的交互来获取信息、验证假设并修正策略。

---

## 一、ReAct 的基本结构

ReAct 模板通常包含以下三种元素的交替序列：

1.  **Thought（思考）**
    
    -   模型对当前状态进行推理，分析已知信息、目标、可能的下一步。
        
    -   示例：“我需要知道巴黎的人口是多少。”
        
2.  **Action（行动）**
    
    -   模型调用一个预定义的工具（如搜索、计算、读取文件等）来获取外部信息。
        
    -   示例：`Search[Paris population]`
        
3.  **Observation（观察）**
    
    -   环境返回行动结果，作为新信息输入给模型。
        
    -   示例：“Observation: The population of Paris is about 2.1 million.”
        

这个过程循环进行，直到任务完成。

> 整个流程类似于人类解决问题的方式：先思考“我缺什么信息？”，再采取行动“去查一下”，然后根据结果继续思考或行动。

---

## 二、ReAct 的工作流程示例

**任务**：判断“埃菲尔铁塔是否比自由女神像高？”

```text
Thought: 我需要分别知道埃菲尔铁塔和自由女神像的高度。
Action: Search[Eiffel Tower height]
Observation: The Eiffel Tower is 330 meters tall (including antenna).

Thought: 现在我知道埃菲尔铁塔高330米，接下来查自由女神像。
Action: Search[Statue of Liberty height]
Observation: The Statue of Liberty is 93 meters tall (from ground to torch tip).

Thought: 330米 > 93米，因此埃菲尔铁塔更高。
Answer: 是的，埃菲尔铁塔比自由女神像高。
```

---

## 三、ReAct 的优势

| 优势  | 说明  |
| --- | --- |
| **减少幻觉（Hallucination）** | 通过外部工具验证事实，避免模型凭空编造答案。 |
| **提升可解释性** | 每一步的思考和行动都显式记录，便于调试和审计。 |
| **支持复杂任务分解** | 能将多跳推理（multi-hop reasoning）任务拆解为多个子步骤。 |
| **动态适应环境** | 可根据观察结果调整策略，实现“试错-修正”机制。 |
| **工具集成能力强** | 易于接入搜索引擎、计算器、代码解释器、数据库等外部工具。 |

---

## 四、ReAct 与传统方法的对比

| 方法  | 特点  | 局限  |
| --- | --- | --- |
| **纯推理（Chain-of-Thought, CoT）** | 仅靠内部逻辑链推理 | 无法获取新信息，易产生错误假设 |
| **纯行动（Act-only）** | 直接调用工具执行 | 缺乏规划，可能执行无效或冗余操作 |
| **ReAct（Reasoning + Acting）** | 交替思考与行动，闭环反馈 | 需要设计合理的工具接口和提示模板 |

---

## 五、实现 ReAct 的关键技术点

1.  **工具定义（Tool Specification）**
    
    -   明确可用的 Action 类型（如 `Search`, `Calculate`, `Lookup` 等）及其输入输出格式。
        
2.  **提示工程（Prompt Engineering）**
    
    -   在系统提示中提供 ReAct 模板和少量示例（few-shot），**引导模型按格式输出**。
        
3.  **执行引擎（Execution Engine）**
    
    -   解析模型输出的 Action，调用对应工具，并将 Observation 拼接回上下文。
        
4.  **终止条件**
    
    -   设定最大步数、检测到 `Answer:` 或任务完成信号时停止循环。
        

---

## 六、应用场景

-   **问答系统**（需多步检索的事实型问题）
    
-   **任务自动化**（如订票、查天气、写报告）
    
-   **代码生成与调试**（结合代码解释器执行并验证）
    
-   **智能代理（AI Agent）**（如 WebAgent、AutoGPT 的底层机制）
    

---

## 七、局限与挑战

-   **依赖高质量工具**：若工具返回错误信息，模型可能被误导。
    
-   **推理成本高**：多轮交互增加延迟和计算开销。
    
-   **格式敏感**：模型需严格遵循 Thought/Action/Observation 格式，否则解析失败。
    
-   **规划能力有限**：对于高度复杂的长期规划任务，仍需更高级的架构（如分层规划、记忆机制）。
    

---

## 八、扩展与演进

-   **Reflexion**：在 ReAct 基础上加入自我反思机制，从失败中学习。
    
-   **Plan-and-Execute**：先生成完整计划，再逐步执行（结合 ReAct 的每步验证）。
    
-   **Toolformer / ToolLLM**：训练模型原生理解工具调用，而非仅靠提示工程。
    

---

## 总结

ReAct 是连接大语言模型“内在推理”与“外在行动”的桥梁。它不仅提升了模型解决现实世界问题的能力，也为构建可信赖、可解释、可交互的 AI Agent 奠定了基础。随着工具生态和模型能力的发展，ReAct 范式正成为智能体架构的核心组件之一。

> 正如论文所言：“**Let language models think and act, not just predict.**”