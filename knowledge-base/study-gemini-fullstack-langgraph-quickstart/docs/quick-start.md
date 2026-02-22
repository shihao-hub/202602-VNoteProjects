# 快速启动指南

## 🚀 一键启动（推荐）

### 前提条件

1. ✅ 已获取 Gemini API Key
2. ✅ 已配置 `backend/.env` 文件
3. ✅ 已安装依赖（前端 + 后端）

### 启动命令

```bash
# 方式 1: 使用 Makefile（推荐）
make dev

# 方式 2: 分别启动
# 终端 1 - 后端
make dev-backend

# 终端 2 - 前端
make dev-frontend
```

### 访问应用

- **前端应用**：http://localhost:5173/app
- **后端 API**：http://localhost:2024
- **LangGraph UI**：http://localhost:2024

---

## 📝 首次运行检查清单

### 步骤 1: 环境检查

```bash
# 检查 Node.js 版本（需要 18+）
node --version
# 输出示例: v24.10.0 ✅

# 检查 Python 版本（需要 3.11+）
python --version
# 输出示例: Python 3.12.5 ✅
```

### 步骤 2: 配置 API Key

1. 访问 [Google AI Studio](https://aistudio.google.com/app/apikey)
2. 创建 API Key
3. 编辑 `backend/.env` 文件：

```bash
# backend/.env
GEMINI_API_KEY=AIzaSy...你的实际Key
```

### 步骤 3: 安装依赖

```bash
# 后端依赖
cd backend
pip install .
cd ..

# 前端依赖
cd frontend
npm install
cd ..
```

### 步骤 4: 启动应用

```bash
make dev
```

### 步骤 5: 测试功能

在浏览器中访问 http://localhost:5173/app，输入问题：

```
What are the latest trends in renewable energy?
```

---

## 🔧 常见启动问题

### 问题 1: 端口已被占用

**错误信息**：
```
Error: listen EADDRINUSE: address already in use :::2024
```

**解决方案**：
```bash
# Windows
netstat -ano | findstr :2024
taskkill /PID <进程ID> /F

# macOS/Linux
lsof -ti:2024 | xargs kill -9
```

### 问题 2: API Key 无效

**错误信息**：
```
Error: Invalid API key
```

**检查清单**：
1. 确认 `backend/.env` 文件存在
2. 确认 API Key 格式正确（以 `AIzaSy` 开头）
3. 确认没有多余的引号或空格

### 问题 3: 依赖安装失败

**后端依赖失败**：
```bash
# 升级 pip
pip install --upgrade pip setuptools wheel

# 重新安装
cd backend
pip install .
```

**前端依赖失败**：
```bash
# 清除缓存
cd frontend
rm -rf node_modules package-lock.json
npm install
```

---

## 📚 更多资源

- 📖 [完整运行指南](./running-guide.md)
- 🔗 [官方文档](https://github.com/langchain-ai/langgraph)
- 💬 [常见问题](./running-guide.md#常见问题)

---

## ✅ 下一步

启动成功后，你可以：

1. **测试基本功能** - 在聊天框中提问
2. **查看 Agent 流程** - 访问 LangGraph UI
3. **自定义配置** - 修改 `backend/src/agent/graph.py`
4. **部署生产环境** - 参考 README.md 部署章节

需要帮助？查看 [完整运行指南](./running-guide.md) 或提交 Issue。
