# Bash Agent 使用指南

> **快速上手**：了解如何使用 Bash Agent 框架

---

## 目录

1. [快速开始](#快速开始)
2. [基础用法](#基础用法)
3. [高级功能](#高级功能)
4. [示例场景](#示例场景)
5. [最佳实践](#最佳实践)
6. [故障排除](#故障排除)
7. [扩展指南](#扩展指南)

---

## 快速开始

### 1. 安装依赖

```bash
# 克隆仓库
git clone <repository-url>
cd bash_is_all_you_need

# 创建虚拟环境
python -m venv venv

# 激活虚拟环境
# Windows
venv\Scripts\activate
# Linux/Mac
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt
```

### 2. 配置环境变量

```bash
# 设置 API Key
export ANTHROPIC_API_KEY="your-api-key-here"

# Windows PowerShell
$env:ANTHROPIC_API_KEY="your-api-key-here"
```

### 3. 运行示例

```python
# 运行邮件分析示例
python examples/email_analyzer.py

# 运行数据处理示例
python examples/data_processor.py
```

### 4. 使用 Agent

```python
from core.agent import BashAgent

# 创建 Agent 实例
agent = BashAgent()

# 执行任务
result = agent.run("计算 data.csv 文件中销售额的总和")

# 查看结果
print(result)
```

---

## 基础用法

### 创建 Agent 实例

```python
from core.agent import BashAgent

# 使用默认配置
agent = BashAgent()

# 使用自定义配置
agent = BashAgent(
    model="claude-3-5-sonnet-20241022",
    work_dir="./my_workspace",
    max_iterations=15
)
```

### 执行任务

#### 简单任务

```python
# 列出文件
agent.run("列出当前目录的所有 Python 文件")

# 执行命令
agent.run("统计 log.txt 中包含 ERROR 的行数")
```

#### 复杂任务

```python
# 多步骤任务
agent.run("""
分析 sales.csv 文件：
1. 计算总销售额
2. 按产品分类统计销售额
3. 找出销售额最高的产品
4. 将结果保存到 sales_report.txt
""")
```

### 访问记忆

```python
# 查看短期记忆（工作区）
memory = agent.memory

# 检索信息
task_goal = memory.retrieve("task_goal")
print(f"任务目标: {task_goal}")

# 搜索记忆
results = memory.search("销售额")
print(f"相关记录: {results}")

# 列出所有文件
import os
for root, dirs, files in os.walk("workspace"):
    for file in files:
        print(os.path.join(root, file))
```

---

## 高级功能

### 1. 自定义配置

编辑 `config.yaml`：

```yaml
agent:
  model: "claude-3-5-sonnet-20241022"
  max_iterations: 20
  work_dir: "./workspace"

llm:
  temperature: 0.5          # 降低温度以获得更确定的输出
  max_tokens: 8192         # 增加 token 限制

memory:
  max_size: 2000          # 增加记忆容量
  retention_days: 30      # 保留记忆 30 天
```

### 2. 安全配置

```yaml
tools:
  safety:
    blacklist:           # 禁止的命令
      - rm -rf
      - dd
      - mkfs
    whitelist_mode: true # 启用白名单模式
    whitelist:           # 只允许这些命令
      - grep
      - find
      - awk
      - sed
      - cat
      - head
      - tail
      - wc
```

### 3. 监控执行

```python
# 启用详细日志
import logging
logging.basicConfig(level=logging.DEBUG)

# 查看执行历史
history = agent.get_execution_history()
for step in history:
    print(f"步骤 {step['iteration']}: {step['command']}")
    print(f"结果: {step['result']}")
    print(f"耗时: {step['duration']}s")
```

### 4. 并行执行

```python
# 并行执行多个独立任务
tasks = [
    "分析 data1.csv",
    "分析 data2.csv",
    "分析 data3.csv"
]

import asyncio
results = asyncio.run(agent.run_parallel(tasks))
```

---

## 示例场景

### 场景 1：日志分析

```python
agent.run("""
分析服务器日志 /var/log/auth.log：

1. 统计登录失败的次数
2. 找出尝试登录最多的 IP 地址
3. 按时间段（小时）统计失败次数
4. 将结果保存到 auth_report.txt
""")
```

**实际执行流程**：

```bash
# Agent 可能执行的命令序列

# 1. 查看日志文件
head -n 20 /var/log/auth.log

# 2. 统计失败次数
grep "Failed password" /var/log/auth.log | wc -l

# 3. 提取 IP 地址并统计
grep "Failed password" /var/log/auth.log | \
  grep -oP 'from \K[\d.]+' | \
  sort | uniq -c | sort -rn > top_ips.txt

# 4. 按小时统计
grep "Failed password" /var/log/auth.log | \
  grep -oP '\d{2}:\d{2}:\d{2}' | \
  cut -d: -f1 | sort | uniq -c > hourly_stats.txt
```

### 场景 2：数据清洗

```python
agent.run("""
清洗 dirty_data.csv 文件：

1. 删除重复行
2. 填充缺失值（数值列用平均值，文本列用 "Unknown"）
3. 标准化日期格式
4. 保存为 cleaned_data.csv
""")
```

### 场景 3：批量文件处理

```python
agent.run("""
处理 data/ 目录下的所有 JSON 文件：

1. 提取每个文件中的 'price' 字段
2. 计算总价格和平均价格
3. 找出价格最高的 10 个文件
4. 生成汇总报告
""")
```

**执行示例**：

```bash
# Agent 执行流程

# 1. 列出所有 JSON 文件
find data/ -name "*.json" > json_files.txt

# 2. 提取所有价格
cat json_files.txt | while read file; do
    jq -r '.price' "$file"
done > all_prices.txt

# 3. 计算总和和平均值
awk '{sum+=$1; count++} END {print "Sum:", sum, "Avg:", sum/count}' all_prices.txt

# 4. 排序找出前 10
sort -rn all_prices.txt | head -n 10 > top10.txt
```

### 场景 4：网站监控

```python
agent.run("""
监控网站 https://example.com：

1. 检查网站是否可访问
2. 获取响应时间
3. 提取页面标题
4. 将结果记录到 monitoring.log
""")
```

---

## 最佳实践

### 1. 任务分解

**❌ 不推荐**：一次执行复杂任务

```python
agent.run("分析所有 CSV 文件，生成图表，并发送邮件")
```

**✅ 推荐**：分步骤执行

```python
# 先分析
agent.run("分析所有 CSV 文件，保存结果到 analysis/")

# 再生成图表
agent.run("根据 analysis/ 的结果生成图表")

# 最后发送
agent.run("将图表发送到 user@example.com")
```

### 2. 保存中间结果

**❌ 不推荐**：不保存中间结果

```bash
# 一次性完成，失败后无法恢复
grep "ERROR" log.txt | awk '{print $5}' | sort | uniq -c | sort -rn
```

**✅ 推荐**：保存中间结果

```bash
# 分步保存，可以回溯和调试
grep "ERROR" log.txt > errors.txt
awk '{print $5}' errors.txt > error_codes.txt
sort error_codes.txt | uniq -c | sort -rn > final_report.txt
```

### 3. 使用管道组合

**✅ 推荐**：利用 Bash 的管道能力

```bash
# 组合多个简单命令
cat data.csv | \
  awk -F, '{print $3}' | \
  sort | \
  uniq -c | \
  sort -rn
```

### 4. 验证每一步

**✅ 推荐**：逐步验证

```python
# 先看看数据长什么样
agent.run("查看 data.csv 的前 5 行")

# 再执行分析
agent.run("计算 data.csv 的行数和列数")

# 最后做复杂分析
agent.run("分析 data.csv 中的趋势")
```

### 5. 错误处理

**✅ 推荐**：让 Agent 自我修正

```python
# Agent 会自动处理错误
agent.run("""
连接数据库 db.example.com，如果连接失败：
1. 检查网络连接
2. 尝试其他主机
3. 检查防火墙设置
""")
```

---

## 故障排除

### 问题 1：命令执行失败

**症状**：Agent 反复执行失败的命令

**解决方案**：

```python
# 方法 1：手动测试命令
bash_executor.execute("your-command")

# 方法 2：查看详细日志
agent.run("列出 workspace/ 目录的所有文件", verbose=True)

# 方法 3：重置并重新开始
agent.reset()
agent.run("你的任务")
```

### 问题 2：内存不足

**症状**：上下文窗口溢出

**解决方案**：

```yaml
# 调整配置
llm:
  max_tokens: 8192  # 增加上下文窗口

memory:
  auto_cleanup: true
  retention_days: 3  # 减少记忆保留时间
```

### 问题 3：安全限制

**症状**：命令被安全检查器阻止

**解决方案**：

```yaml
# 1. 放宽白名单
tools:
  safety:
    whitelist_mode: false  # 关闭白名单模式

# 2. 添加命令到白名单
whitelist:
  - grep
  - awk
  - jq
  - your-custom-command
```

### 问题 4：性能问题

**症状**：执行速度慢

**解决方案**：

```python
# 1. 减少迭代次数
agent = BashAgent(max_iterations=5)

# 2. 启用缓存
agent.enable_help_cache()

# 3. 并行执行
results = agent.run_parallel(tasks)
```

---

## 扩展指南

### 1. 添加自定义工具

```python
# core/tools.py
def custom_tool(arg1: str, arg2: int) -> str:
    """自定义工具"""
    # 实现你的逻辑
    return result

# 注册工具
agent.register_tool("custom_tool", custom_tool)
```

### 2. 集成外部 API

```python
# 使用 curl 调用 API
agent.run("""
使用 curl 调用 GitHub API：
1. 获取用户信息
2. 列出公开仓库
3. 统计 stars 总数
""")

# Agent 会执行：
# curl https://api.github.com/users/username
# curl https://api.github.com/users/username/repos
```

### 3. 添加新的记忆类型

```python
# core/memory.py
class VectorMemory(FileSystemMemory):
    """添加向量搜索能力"""

    def vector_search(self, query: str, top_k: int = 5):
        """使用向量搜索"""
        # 实现向量搜索
        pass
```

### 4. 自定义提示词

```python
# 创建自定义提示词
custom_prompt = """
你是一个专业的数据分析助手。

你的专长：
- 数据清洗和预处理
- 统计分析
- 数据可视化

工作流程：
1. 理解数据结构
2. 执行分析
3. 生成报告
"""

agent = BashAgent(system_prompt=custom_prompt)
```

---

## 参考资源

- **架构文档**：`architecture.md`
- **理论分析**：`llm-agent-architecture.md`
- **Bash 方案**：`bash-is-all-you-need-analysis.md`
- **示例代码**：`examples/` 目录

---

## 常见问题 (FAQ)

### Q1: Bash Agent 和传统 Agent 框架有什么区别？

**A**: Bash Agent 使用文件系统作为记忆，Bash 命令作为工具，相比传统框架更灵活、可扩展性更强。详见 `bash-vs-general-agent.md`。

### Q2: 适合哪些场景？

**A**:
- ✅ 技术型任务（数据处理、日志分析、系统管理）
- ✅ 需要组合多个工具的复杂任务
- ✅ 原型开发和快速迭代
- ❌ 非技术用户的日常任务（建议使用专用工具）

### Q3: 如何降低 Token 成本？

**A**:
1. 预定义常用工具，减少 --help 查询
2. 使用帮助文档缓存
3. 分解任务，避免过长的上下文
4. 使用更小的模型（如 Claude Haiku）

### Q4: 安全性如何保障？

**A**:
1. 命令黑名单/白名单
2. 沙箱执行环境（Docker/firejail）
3. 资源限制（CPU、内存、磁盘）
4. 详细的审计日志

---

## 更新日志

- **v0.1.0** (2026-01-08): 初始版本
  - 核心框架实现
  - 基础示例
  - 文档完善
