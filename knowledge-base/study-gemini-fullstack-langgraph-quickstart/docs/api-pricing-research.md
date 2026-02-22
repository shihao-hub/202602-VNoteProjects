# API 收费情况调研

本文档记录了项目中所使用的 API 的收费情况，包括 Google Gemini API 和 LangSmith。

## Google Gemini API 收费情况

### 按 Token 计费（每百万 Token）

- **Input（输入）**: $0.10 - $4.00
- **Output（输出）**: $0.40 - $18.00

### 主要模型定价

| 模型 | Input 价格 | Output 价格 |
|------|-----------|-------------|
| Gemini 3 Flash Preview | 免费 | 免费 |
| Gemini 3.0 Flash | $0.50 | $3.00 |
| Gemini 3.0 Pro | $2-4 | $12-18 |

### 免费额度

- 每日一定额度的免费调用
- 最多 100 万 token 上下文窗口
- 无需信用卡

### 额外功能

- **Google Search 接入**: 每天 1,500 次免费，之后 $35/1,000 次请求

---

## LangSmith 收费情况

### Developer Plan（免费版）

- **价格**: $0
- **包含**: 1 个席位
- **额度**: 5,000 traces/月
- **数据保留**: 14 天

### Plus Plan（付费版）

- **价格**: $39/用户/月
- **席位**: 最多 10 个
- **额度**: 10,000 traces/月
- **功能**: 高级调试和评估工具

### 按量付费

- 前 5,000 traces 免费
- 超出后：**$0.50 / 1,000 traces**

### Enterprise Plan

- 自定义定价
- 支持私有化部署
- 适合大型团队

---

## 💰 成本建议

对于学习/个人项目：

- **Gemini API**: 使用免费额度或 Flash 模型，成本极低
- **LangSmith**: 免费版（5,000 traces/月）完全够用

**重要提示**: LangSmith 的 trace 是指每次 LangGraph 运行记录，不是按 token 计费。一般学习项目很难超过 5,000 traces/月的限制。

---

## 参考链接

- [Gemini Developer API pricing](https://ai.google.dev/gemini-api/docs/pricing)
- [LangSmith Plans and Pricing](https://www.langchain.com/pricing)
- [LangSmith Pricing (2026)](https://agentsapis.com/langsmith-pricing/)

---

*最后更新时间: 2026-02-07*
