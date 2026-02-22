# Gemini Fullstack LangGraph Quickstart 运行方案

## 项目概述

这是一个基于 **React 前端 + LangGraph 后端** 的全栈应用，演示了如何使用 Google Gemini 模型构建智能研究助手。该应用能够：
- 动态生成搜索查询
- 使用 Google Search API 进行网络研究
- 反思并识别知识差距
- 提供带引用的答案

## 技术栈

### 前端
- **React 19.0.0** - UI 框架
- **Vite** - 构建工具
- **TypeScript** - 类型安全
- **Tailwind CSS 4.1.5** - 样式框架
- **Radix UI** - UI 组件库
- **LangGraph SDK** - 与后端通信

### 后端
- **Python 3.11+** - 运行时环境
- **LangGraph 0.2.6+** - AI 代理框架
- **LangChain 0.3.19+** - LLM 集成
- **FastAPI** - Web 框架
- **Google Gemini** - 大语言模型

---

## 📋 运行步骤

### 步骤 1：获取 Gemini API Key（必需）

1. 访问 [Google AI Studio](https://aistudio.google.com/app/apikey)
2. 登录 Google 账号
3. 点击 **"Create API Key"** 创建新的 API Key
4. 复制生成的 API Key（格式：`AIzaSy...`）
5. **保存好这个 Key**，稍后配置时会用到

> 💡 **提示**：新用户通常可以获得免费额度

---

### 步骤 2：安装依赖

#### 2.1 检查环境要求

```bash
# 检查 Node.js 版本（需要 18+）
node --version

# 检查 Python 版本（需要 3.11+）
python --version
# 或
python3 --version
```

如果环境不符合要求，请先安装：
- **Node.js**: 从 [nodejs.org](https://nodejs.org/) 下载 LTS 版本
- **Python**: 从 [python.org](https://www.python.org/) 下载 3.11 或更高版本

#### 2.2 安装前端依赖

```bash
cd frontend
npm install
cd ..
```

> **预计时间**：2-5 分钟（取决于网络速度）

#### 2.3 安装后端依赖

```bash
cd backend
pip install .
cd ..
```

> **预计时间**：3-8 分钟（首次安装较慢）
>
> **建议**：使用虚拟环境（可选但推荐）
> ```bash
> # Windows
> python -m venv venv
> venv\Scripts\activate
>
> # macOS/Linux
> python3 -m venv venv
> source venv/bin/activate
> ```

---

### 步骤 3：配置环境变量

1. 在 `backend/` 目录下创建 `.env` 文件：

```bash
# 从示例文件复制
cp backend/.env.example backend/.env
```

2. 编辑 `.env` 文件，添加你的 API Key：

```bash
# backend/.env
GEMINI_API_KEY=你的实际API_Key
```

**示例**：
```bash
GEMINI_API_KEY=AIzaSyABC123xyzXYZ789...
```

---

### 步骤 4：启动应用

#### 方式 A：一键启动（推荐）

```bash
make dev
```

这会同时启动前端和后端开发服务器。

#### 方式 B：分别启动（适合调试）

**终端 1 - 启动后端**：
```bash
make dev-backend
# 或
cd backend && langgraph dev
```

**终端 2 - 启动前端**：
```bash
make dev-frontend
# 或
cd frontend && npm run dev
```

---

### 步骤 5：访问应用

启动成功后，在浏览器中打开：

- **前端应用**：http://localhost:5173/app
- **后端 API**：http://localhost:2024
- **LangGraph UI**：http://localhost:2024（会自动打开）

---

## ✅ 验证安装

### 1. 检查前端

访问 http://localhost:5173/app，应该看到：
- 欢迎界面
- 聊天输入框
- "Ask a question to get started" 提示

### 2. 检查后端

访问 http://localhost:2024，应该看到：
- LangGraph Studio 界面
- Agent 图表可视化

### 3. 功能测试

在聊天框中输入一个问题，例如：
```
What are the latest trends in renewable energy?
```

系统应该：
- 生成搜索查询
- 显示搜索进度
- 提供带引用的答案

---

## 📁 关键文件说明

| 文件路径 | 用途 |
|---------|------|
| `backend/.env` | 环境变量配置（需手动创建） |
| `backend/src/agent/graph.py` | LangGraph 核心逻辑 |
| `backend/src/agent/app.py` | FastAPI 应用入口 |
| `frontend/src/App.tsx` | React 主应用（含 API 配置） |
| `Makefile` | 开发命令快捷方式 |

---

## 🔧 常见问题

### Q1: `pip install .` 失败

**解决方案**：
```bash
# 尝试升级 pip
pip install --upgrade pip setuptools wheel

# 重新安装
cd backend
pip install .
```

### Q2: 前端无法连接后端

**检查**：
1. 后端是否正常运行在 `http://localhost:2024`
2. 查看 `frontend/src/App.tsx` 中的 `apiUrl` 配置
3. 检查浏览器控制台是否有 CORS 错误

### Q3: API Key 无效

**检查**：
1. 确认 `.env` 文件在 `backend/` 目录下
2. 确认 API Key 格式正确（以 `AIzaSy` 开头）
3. 确认 API Key 已启用（访问 Google AI Console 检查）

### Q4: Python 版本不兼容

**解决方案**：
```bash
# 检查版本
python --version

# 如果低于 3.11，需要升级
# Windows: 从 python.org 下载安装
# macOS: brew install python@3.11
# Ubuntu: sudo apt install python3.11
```

---

## 🚀 开发体验

### 热重载
- **前端**：修改代码后自动刷新（Vite）
- **后端**：修改代码后自动重启（Uvicorn）

### 生产构建

```bash
# 构建前端
cd frontend
npm run build

# 运行 Docker Compose（生产环境）
cd ..
docker-compose up
```

生产环境访问：http://localhost:8123/app/

---

## 📚 其他功能

### CLI 示例（快速测试）

```bash
cd backend
python examples/cli_research.py "What are the latest trends in renewable energy?"
```

这会直接在终端运行代理并打印结果，无需启动前端。

---

## 🎯 下一步

1. **获取 API Key** - 访问 Google AI Studio
2. **安装依赖** - 前端和后端
3. **配置环境** - 创建 `.env` 文件
4. **启动应用** - 运行 `make dev`
5. **测试功能** - 在浏览器中提问

---

## ⚠️ 注意事项

- **API Key 安全**：不要将 `.env` 文件提交到 Git
- **成本控制**：留意 Gemini API 的使用配额和费用
- **网络要求**：需要稳定的网络连接（访问 Google API）
- **端口占用**：确保 2024 和 5173 端口未被占用

---

**准备就绪？** 执行上述步骤，如有问题随时询问！
