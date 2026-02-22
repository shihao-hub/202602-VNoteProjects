# Bash Agent 框架架构设计

> **设计理念**：基于 "Bash Is All You Need" 和 `Agent = LLM + Planning + Memory + Tools` 的通用架构

---

## 整体架构

### 架构层次图

```
┌─────────────────────────────────────────────────────────────┐
│                        用户层 (User)                         │
│                    用户输入任务/目标                           │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                      Agent 控制层                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Agent 主循环 (ReAct Pattern)                          │  │
│  │  Observe → Thought → Act → Result → Loop             │  │
│  └──────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
         ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  LLM Engine  │ │   Memory     │ │    Tools     │
│  (大脑)      │ │  (记忆)       │ │  (手脚)       │
├──────────────┤ ├──────────────┤ ├──────────────┤
│ Claude/GPT   │ │ Filesystem   │ │ Bash Executor│
│ Reasoning   │ │ - Short-term │ │ Tool Discovery│
│ Planning    │ │ - Long-term  │ │ --help query  │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## 核心组件详解

### 1. Agent 主循环 (core/agent.py)

**职责**：实现 ReAct (Reason + Act) 模式的执行循环

```python
class BashAgent:
    """
    Bash Agent 主控制器

    Agent = LLM + Planning + Memory + Tools
    """

    def run(self, user_query: str):
        """
        主执行循环

        Loop:
        1. Observe (观察)：获取用户输入和当前状态
        2. Thought (思考)：LLM 分析情况，制定计划
        3. Act (行动)：执行 Bash 命令
        4. Result (结果)：获取执行结果
        5. Repeat (循环)：直到任务完成
        """
        while not self.is_done():
            # 观察
            observation = self.observe()

            # 思考（调用 LLM）
            thought = self.think(observation)

            # 行动（执行 Bash）
            action_result = self.act(thought.command)

            # 更新记忆
            self.memory.store(action_result)

            # 判断是否完成
            if self.check_done():
                break
```

**关键特性：**
- **状态管理**：跟踪当前任务状态
- **错误处理**：失败时重新思考和调整
- **记忆集成**：自动保存重要信息到文件系统
- **工具选择**：决定何时查询 --help，何时执行命令

---

### 2. LLM 引擎集成

**模型选择**：
- **推荐**：Claude 3.5 Sonnet (平衡性能和成本)
- **备选**：GPT-4, Claude Opus (更强大但更贵)

**提示词策略**：

```python
SYSTEM_PROMPT = """
你是一个可以使用 Bash 命令完成任务的智能助手。

你的能力：
1. 执行任意 Bash 命令
2. 读写文件系统
3. 使用 --help 查询工具用法
4. 组合多个命令完成复杂任务

工作流程：
1. 理解用户的目标
2. 分解为可执行的步骤
3. 执行 Bash 命令
4. 验证结果
5. 如果失败，分析原因并重试

注意事项：
- 优先使用简单命令
- 复杂操作分步执行
- 保存中间结果到文件
- 遇到未知工具，使用 --help 查询
"""
```

---

### 3. Bash 执行器 (core/bash_executor.py)

**职责**：安全地执行 Bash 命令并返回结果

```python
class BashExecutor:
    """
    Bash 命令执行器

    功能：
    - 执行命令并捕获输出
    - 处理错误和超时
    - 支持管道操作
    - 保存中间结果到文件
    """

    def execute(self, command: str, timeout: int = 30) -> ExecutionResult:
        """
        执行 Bash 命令

        Args:
            command: 要执行的命令
            timeout: 超时时间（秒）

        Returns:
            ExecutionResult:
                - stdout: 标准输出
                - stderr: 标准错误
                - return_code: 返回码
                - success: 是否成功
        """
        # 安全检查
        if not self.is_safe(command):
            raise UnsafeCommandError(command)

        # 执行命令
        result = subprocess.run(
            command,
            shell=True,
            capture_output=True,
            text=True,
            timeout=timeout
        )

        return ExecutionResult(
            stdout=result.stdout,
            stderr=result.stderr,
            return_code=result.returncode,
            success=result.returncode == 0
        )

    def save_output(self, output: str, filepath: str):
        """保存输出到文件（外部记忆）"""
        with open(filepath, 'w') as f:
            f.write(output)
```

**安全特性：**
- 命令白名单/黑名单
- 沙箱执行环境（可选）
- 超时保护
- 资源限制（CPU, 内存）

---

### 4. 记忆管理 (core/memory.py)

**职责**：管理文件系统记忆（短期 + 长期）

```python
class FileSystemMemory:
    """
    基于文件系统的记忆管理

    映射到通用架构：
    - 短期记忆：workspace/ (当前任务上下文)
    - 长期记忆：docs/, history/ (持久化知识)
    """

    def __init__(self, work_dir: str = "./workspace"):
        self.work_dir = Path(work_dir)
        self.work_dir.mkdir(exist_ok=True)

        # 短期记忆目录
        self.context_dir = self.work_dir / "context"
        self.results_dir = self.work_dir / "results"
        self.temp_dir = self.work_dir / "temp"

        for dir in [self.context_dir, self.results_dir, self.temp_dir]:
            dir.mkdir(exist_ok=True)

    def store(self, key: str, value: str, category: str = "context"):
        """
        存储信息到文件系统

        Args:
            key: 信息键
            value: 信息内容
            category: 类别 (context/results/temp)
        """
        dir_map = {
            "context": self.context_dir,
            "results": self.results_dir,
            "temp": self.temp_dir
        }

        filepath = dir_map[category] / f"{key}.txt"
        with open(filepath, 'w', encoding='utf-8') as f:
            f.write(value)

        return filepath

    def retrieve(self, key: str, category: str = "context") -> Optional[str]:
        """检索信息"""
        filepath = self.work_dir / category / f"{key}.txt"
        if filepath.exists():
            return filepath.read_text(encoding='utf-8')
        return None

    def search(self, pattern: str) -> List[str]:
        """
        搜索记忆

        使用 grep 搜索所有记忆文件
        """
        result = subprocess.run(
            f"grep -r '{pattern}' {self.work_dir}",
            shell=True,
            capture_output=True,
            text=True
        )
        return result.stdout.split('\n') if result.stdout else []
```

**目录结构**：

```
workspace/
├── context/              # 短期记忆（当前任务）
│   ├── task_goal.txt     # 任务目标
│   ├── progress.txt      # 当前进度
│   └── user_input.txt    # 用户输入
├── results/             # 执行结果
│   ├── analysis.txt
│   └── output.txt
└── temp/               # 临时文件
    └── temp_data.txt
```

---

### 5. 工具发现 (core/tools.py)

**职责**：渐进式上下文披露，按需查询工具信息

```python
class ToolDiscovery:
    """
    工具发现和管理

    实现"渐进式上下文披露"：
    1. 启动时只加载常用工具
    2. 需要时查询 --help
    3. 缓存帮助文档
    """

    def __init__(self):
        self.help_cache = {}
        self.common_commands = {
            'grep': '搜索文本',
            'find': '查找文件',
            'awk': '文本处理',
            'sed': '文本替换',
            'jq': 'JSON 处理',
            'curl': 'HTTP 请求'
        }

    def get_help(self, command: str) -> str:
        """
        获取命令的帮助信息

        优先从缓存读取，缓存未命中则执行 --help
        """
        if command in self.help_cache:
            return self.help_cache[command]

        # 执行 --help
        result = subprocess.run(
            f"{command} --help",
            shell=True,
            capture_output=True,
            text=True
        )

        help_text = result.stdout or result.stderr
        self.help_cache[command] = help_text

        return help_text

    def suggest_tools(self, task: str) -> List[str]:
        """
        根据任务推荐工具

        基于任务描述推荐合适的 Bash 命令
        """
        suggestions = []

        if "搜索" in task or "查找" in task:
            suggestions.extend(['grep', 'find'])

        if "JSON" in task:
            suggestions.append('jq')

        if "下载" in task or "请求" in task:
            suggestions.append('curl')

        return suggestions
```

---

## 数据流图

### 任务执行流程

```
用户输入："计算 CSV 文件中每列的平均值"
    │
    ▼
┌─────────────────────────────────────────┐
│ Agent.observe()                         │
│   - 读取用户输入                        │
│   - 检查记忆中是否有相关上下文           │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Agent.think() → LLM                     │
│   - 分析任务：需要处理 CSV              │
│   - 制定计划：awk 计算平均值            │
│   - 输出：使用 awk 命令                 │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Agent.act() → BashExecutor             │
│   - 执行：awk -F, '{for(i=1;i<=NF;i++)  │
│              sum[i]+=$i}                │
│              END{for(i=1;i<=NF;i++)     │
│              print i, sum[i]/NR}' data.csv│
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ ExecutionResult                        │
│   - stdout: "1 50.3\n2 75.2\n..."      │
│   - stderr: ""                         │
│   - success: True                      │
└────────────┬────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────┐
│ Memory.store()                         │
│   - 保存到 workspace/results/avg.txt    │
└─────────────────────────────────────────┘
             │
             ▼
        返回给用户
```

---

## 配置系统

### config.yaml

```yaml
# Agent 配置
agent:
  model: "claude-3-5-sonnet-20241022"  # 使用的模型
  max_iterations: 10                    # 最大迭代次数
  timeout: 30                          # 命令超时时间（秒）
  work_dir: "./workspace"             # 工作目录

# LLM 配置
llm:
  api_key: "${ANTHROPIC_API_KEY}"      # 环境变量
  temperature: 0.7                     # 温度参数
  max_tokens: 4096                     # 最大 token 数

# 记忆配置
memory:
  max_size: 1000                      # 最大记忆条目数
  retention_days: 7                   # 记忆保留天数
  auto_cleanup: true                  # 自动清理过期记忆

# 工具配置
tools:
  help_cache: true                    # 缓存 --help 结果
  common_commands:                    # 常用命令列表
    - grep
    - find
    - awk
    - sed
    - jq
    - curl

  # 安全配置
  safety:
    blacklist:                       # 黑名单命令
      - rm -rf /
      - mkfs
      - dd
      - chmod 000
    whitelist_mode: false            # 白名单模式（更严格）
    whitelist: []                    # 白名单命令
```

---

## 安全设计

### 1. 命令安全

```python
class SecurityChecker:
    """命令安全检查器"""

    DANGEROUS_COMMANDS = [
        'rm -rf /',
        'mkfs',
        'dd if=/dev/zero',
        'chmod 000',
        ':(){ :|:& };:',  # fork bomb
    ]

    def is_safe(self, command: str) -> bool:
        """检查命令是否安全"""
        # 黑名单检查
        for dangerous in self.DANGEROUS_COMMANDS:
            if dangerous in command:
                return False

        # 路径检查
        if '../' in command and 'rm' in command:
            return False

        return True
```

### 2. 资源限制

```python
# 限制执行时间和资源
def execute_with_limits(command: str):
    """
    带资源限制的执行
    - CPU 时间限制
    - 内存限制
    - 磁盘写入限制
    """
    resource.setrlimit(resource.RLIMIT_CPU, (10, 10))  # 10秒 CPU
    resource.setrlimit(resource.RLIMIT_AS, (1024*1024*100, 1024*1024*100))  # 100MB
    # ... 执行命令
```

### 3. 沙箱环境（可选）

```bash
# 使用 Docker 或 firejail 沙箱
docker run --rm -v $(pwd):/workspace alpine:latest bash -c "command"

# 或使用 firejail
firejail --private --whitelist=workspace bash -c "command"
```

---

## 扩展性设计

### 1. 插件系统（可选）

```python
class Plugin:
    """插件基类"""
    def on_command_start(self, command: str): pass
    def on_command_end(self, result): pass

class LoggingPlugin(Plugin):
    def on_command_start(self, command: str):
        logger.info(f"执行命令: {command}")

    def on_command_end(self, result):
        logger.info(f"命令结果: {result.success}")
```

### 2. 自定义工具

```python
class CustomTool:
    """自定义工具"""

    @staticmethod
    def process_csv(file_path: str) -> str:
        """专门的 CSV 处理工具"""
        import pandas as pd
        df = pd.read_csv(file_path)
        return df.describe().to_string()

# 注册到 Agent
agent.register_tool("process_csv", CustomTool.process_csv)
```

---

## 性能优化

### 1. 帮助文档缓存

```python
class HelpCache:
    """--help 结果缓存"""
    def __init__(self, cache_file: str = ".help_cache.json"):
        self.cache_file = cache_file
        self.cache = self.load_cache()

    def get(self, command: str) -> Optional[str]:
        return self.cache.get(command)

    def set(self, command: str, help_text: str):
        self.cache[command] = help_text
        self.save_cache()
```

### 2. 并行执行

```python
async def execute_parallel(commands: List[str]) -> List[ExecutionResult]:
    """并行执行多个命令"""
    tasks = [execute_async(cmd) for cmd in commands]
    return await asyncio.gather(*tasks)
```

### 3. 结果流式传输

```python
def execute_stream(command: str):
    """流式读取输出"""
    process = subprocess.Popen(
        command,
        shell=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
        text=True
    )

    for line in process.stdout:
        yield line  # 实时返回输出
```

---

## 测试策略

### 1. 单元测试

```python
def test_bash_executor():
    executor = BashExecutor()
    result = executor.execute("echo 'Hello'")
    assert result.success
    assert result.stdout == "Hello\n"

def test_memory_store():
    memory = FileSystemMemory()
    memory.store("test", "content")
    assert memory.retrieve("test") == "content"
```

### 2. 集成测试

```python
def test_agent_workflow():
    agent = BashAgent()
    result = agent.run("列出当前目录的文件")
    assert "result" in result
    assert agent.memory.exists("result")
```

### 3. 安全测试

```python
def test_security_checker():
    checker = SecurityChecker()
    assert not checker.is_safe("rm -rf /")
    assert checker.is_safe("ls -la")
```

---

## 参考文档

- `bash-vs-general-agent.md` - Bash Agent 在通用架构中的定位
- `usage-guide.md` - 使用指南
- `llm-agent-architecture.md` - 通用 Agent 架构理论
