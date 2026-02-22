# 阶段 5: 前后端通信与部署

## 📚 学习目标

- 理解前后端通信机制
- 掌握 LangGraph API 的配置
- 了解部署和生产环境配置
- 理解 Redis 和 PostgreSQL 的作用

---

## 🎯 FastAPI 集成

### app.py 文件分析

**文件位置**: `backend/src/agent/app.py`

### 完整代码

```python
from fastapi import FastAPI, Response
from fastapi.staticfiles import StaticFiles
import pathlib

# 定义 FastAPI 应用
app = FastAPI()

def create_frontend_router(build_dir="../frontend/dist"):
    """创建路由以服务 React 前端

    Args:
        build_dir: React 构建目录相对于此文件的路径

    Returns:
        服务前端的 Starlette 应用
    """
    build_path = pathlib.Path(__file__).parent.parent.parent / build_dir

    # 检查构建目录是否存在
    if not build_path.is_dir() or not (build_path / "index.html").is_file():
        print(
            f"WARN: Frontend build directory not found or incomplete at {build_path}. "
            "Serving frontend will likely fail."
        )
        # 返回一个虚拟路由
        from starlette.routing import Route

        async def dummy_frontend(request):
            return Response(
                "Frontend not built. Run 'npm run build' in the frontend directory.",
                media_type="text/plain",
                status_code=503,
            )

        return Route("/{path:path}", endpoint=dummy_frontend)

    return StaticFiles(directory=build_path, html=True)

# 挂载前端到 /app 路径,不与 LangGraph API 路由冲突
app.mount(
    "/app",
    create_frontend_router(),
    name="frontend",
)
```

### 关键点

| 元素 | 说明 |
|------|------|
| `FastAPI()` | 创建 FastAPI 应用实例 |
| `StaticFiles` | 服务静态文件(HTML, CSS, JS) |
| `app.mount()` | 将前端挂载到 `/app` 路径 |
| `html=True` | 支持 SPA 路由,所有路径返回 index.html |

### 路由结构

```
http://localhost:8123
    ├── /                    → LangGraph API (404, 未定义根路由)
    ├── /runs               → LangGraph API endpoint
    ├── /threads            → LangGraph API endpoint
    ├── /app                → React 前端 (StaticFiles)
    │   ├── /app/           → index.html
    │   ├── /app/assets/    → CSS, JS 文件
    │   └── /*              → index.html (SPA fallback)
    └── /docs               → FastAPI 自动生成的 API 文档
```

---

## 🌐 API 配置

### 开发环境 vs 生产环境

| 配置项 | 开发环境 | 生产环境 |
|--------|---------|---------|
| **前端地址** | `http://localhost:5173` | 由后端服务 (`/app`) |
| **后端地址** | `http://localhost:2024` | `http://localhost:8123` |
| **前端服务器** | Vite Dev Server | 静态文件 (FastAPI) |
| **后端服务器** | LangGraph Dev Server | LangGraph API Server |
| **热重载** | ✅ 支持 | ❌ 不支持 |
| **Redis** | ❌ 不需要 | ✅ 需要 |
| **PostgreSQL** | ❌ 不需要 | ✅ 需要 |

### 前端 apiUrl 配置

**文件位置**: `frontend/src/App.tsx:25-27`

```typescript
apiUrl: import.meta.env.DEV
  ? "http://localhost:2024"   // 开发环境
  : "http://localhost:8123",  // 生产环境
```

**`import.meta.env.DEV`**: Vite 提供的环境变量,开发时为 `true`

---

## 🐳 Docker 部署

### Dockerfile 分析

**文件位置**: 项目根目录 `Dockerfile`

#### 多阶段构建

```dockerfile
# Stage 1: 构建 React 前端
FROM node:20-alpine AS frontend-builder

WORKDIR /app/frontend

# 安装依赖
COPY frontend/package.json ./
COPY frontend/package-lock.json ./
RUN npm install

# 复制源代码并构建
COPY frontend/ ./
RUN npm run build

# Stage 2: Python 后端
FROM docker.io/langchain/langgraph-api:3.11

# 安装 UV (快速 Python 包管理器)
RUN apt-get update && apt-get install -y curl && \
    curl -LsSf https://astral.sh/uv/install.sh | sh && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

ENV PATH="/root/.local/bin:$PATH"

# 从前端构建阶段复制构建文件
COPY --from=frontend-builder /app/frontend/dist /deps/frontend/dist

# 复制后端代码
ADD backend/ /deps/backend

# 安装 Python 依赖
RUN uv pip install --system pip setuptools wheel
RUN cd /deps/backend && \
    PYTHONDONTWRITEBYTECODE=1 UV_SYSTEM_PYTHON=1 \
    uv pip install --system -c /api/constraints.txt -e .

# 设置 LangGraph 环境变量
ENV LANGGRAPH_HTTP='{"app": "/deps/backend/src/agent/app.py:app"}'
ENV LANGSERVE_GRAPHS='{"agent": "/deps/backend/src/agent/graph.py:graph"}'

# 确保 langgraph-api 包不被覆盖
RUN mkdir -p /api/langgraph_api /api/langgraph_runtime /api/langgraph_license /api/langgraph_storage
RUN touch /api/langgraph_api/__init__.py /api/langgraph_runtime/__init__.py /api/langgraph_license/__init__.py /api/langgraph_storage/__init__.py
RUN PYTHONDONTWRITEBYTECODE=1 pip install --no-cache-dir --no-deps -e /api

WORKDIR /deps/backend
```

#### 关键点

1. **多阶段构建**: 前端和后端分开构建
2. **前端构建**: 使用 `node:20-alpine` 镜像
3. **后端基础镜像**: `langchain/langgraph-api:3.11`
4. **UV 包管理器**: 比 pip 快 10-100 倍
5. **环境变量**: 配置 LangGraph 应用和图

### 构建和运行

#### 1. 构建 Docker 镜像

```bash
docker build -t gemini-fullstack-langgraph -f Dockerfile .
```

#### 2. 使用 docker-compose 运行

```bash
GEMINI_API_KEY=<your_key> LANGSMITH_API_KEY=<your_key> docker-compose up
```

---

## 📦 docker-compose.yml 分析

**文件位置**: 项目根目录 `docker-compose.yml`

### 完整配置

```yaml
volumes:
  langgraph-data:
    driver: local

services:
  # Redis 服务
  langgraph-redis:
    image: docker.io/redis:6
    container_name: langgraph-redis
    healthcheck:
      test: redis-cli ping
      interval: 5s
      timeout: 1s
      retries: 5

  # PostgreSQL 服务
  langgraph-postgres:
    image: docker.io/postgres:16
    container_name: langgraph-postgres
    ports:
      - "5433:5432"
    environment:
      POSTGRES_DB: postgres
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - langgraph-data:/var/lib/postgresql/data
    healthcheck:
      test: pg_isready -U postgres
      start_period: 10s
      timeout: 1s
      retries: 5
      interval: 5s

  # LangGraph API 服务
  langgraph-api:
    image: gemini-fullstack-langgraph
    container_name: langgraph-api
    ports:
      - "8123:8000"
    depends_on:
      langgraph-redis:
        condition: service_healthy
      langgraph-postgres:
        condition: service_healthy
    environment:
      GEMINI_API_KEY: ${GEMINI_API_KEY}
      LANGSMITH_API_KEY: ${LANGSMITH_API_KEY}
      REDIS_URI: redis://langgraph-redis:6379
      POSTGRES_URI: postgres://postgres:postgres@langgraph-postgres:5432/postgres?sslmode=disable
```

### 服务详解

#### 1. Redis

| 配置 | 值 | 说明 |
|------|-----|------|
| 镜像 | `redis:6` | Redis 6 |
| 健康检查 | `redis-cli ping` | 每 5 秒检查一次 |
| 端口 | 6379 (内部) | 默认 Redis 端口 |

**作用**: Pub-sub 消息代理,实现实时流式输出

#### 2. PostgreSQL

| 配置 | 值 | 说明 |
|------|-----|------|
| 镜像 | `postgres:16` | PostgreSQL 16 |
| 端口映射 | `5433:5432` | 主机 5433 → 容器 5432 |
| 数据库 | `postgres` | 默认数据库 |
| 用户 | `postgres` | 默认用户 |
| 密码 | `postgres` | 默认密码 |
| 数据卷 | `langgraph-data` | 持久化数据 |

**作用**: 存储状态、任务队列、线程历史

#### 3. LangGraph API

| 配置 | 值 | 说明 |
|------|-----|------|
| 镜像 | `gemini-fullstack-langgraph` | 自定义构建镜像 |
| 端口映射 | `8123:8000` | 主机 8123 → 容器 8000 |
| 依赖 | redis, postgres | 等待它们健康后再启动 |

**环境变量**:
```bash
GEMINI_API_KEY=<your_key>
LANGSMITH_API_KEY=<your_key>
REDIS_URI=redis://langgraph-redis:6379
POSTGRES_URI=postgres://postgres:postgres@langgraph-postgres:5432/postgres?sslmode=disable
```

---

## 🔄 Redis 和 PostgreSQL 的作用

### Redis: Pub-Sub 消息代理

#### 为什么需要 Redis?

在开发环境中,LangGraph 使用内存存储,无需外部依赖。但在生产环境中:

1. **实时流式输出**: 后台任务运行时,需要实时推送事件到前端
2. **Pub-Sub 模式**: Redis 的发布-订阅机制非常适合这种场景
3. **高性能**: 内存数据库,延迟低

#### 工作流程

```
前端 (WebSocket)
    ↑
    │ SSE 事件流
    │
LangGraph API
    ↑
    │ 订阅事件
    │
Redis (Pub-Sub)
    ↑
    │ 发布事件
    │
后台任务 (StateGraph)
```

### PostgreSQL: 状态持久化

#### 为什么需要 PostgreSQL?

1. **线程存储**: 保存对话历史和状态
2. **任务队列**: 管理后台任务的执行
3. **Exactly-Once 语义**: 确保任务只执行一次
4. **检查点**: 保存图的中间状态,支持恢复

#### 数据模型

```sql
-- 线程表
threads (
    id UUID PRIMARY KEY,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    -- ...
)

-- 运行表
runs (
    id UUID PRIMARY KEY,
    thread_id UUID REFERENCES threads(id),
    status TEXT,  -- pending, running, completed, failed
    -- ...
)

-- 状态表
checkpoint (
    id UUID PRIMARY KEY,
    thread_id UUID,
    run_id UUID,
    state JSONB,  -- 存储完整的状态对象
    -- ...
)
```

---

## 🚀 部署流程

### 开发环境启动

```bash
# 终端 1: 启动后端
cd backend
langgraph dev

# 终端 2: 启动前端
cd frontend
npm run dev
```

**访问**:
- 前端: http://localhost:5173/app
- 后端 API: http://localhost:2024
- LangGraph UI: http://localhost:2024 (自动打开)

### 生产环境部署

#### 1. 构建前端

```bash
cd frontend
npm run build
```

**输出**: `frontend/dist/` 目录

#### 2. 构建 Docker 镜像

```bash
cd project-root
docker build -t gemini-fullstack-langgraph -f Dockerfile .
```

#### 3. 启动 docker-compose

```bash
# 设置环境变量
export GEMINI_API_KEY="your_key"
export LANGSMITH_API_KEY="your_key"

# 启动服务
docker-compose up -d
```

#### 4. 访问应用

- 前端: http://localhost:8123/app
- 后端 API: http://localhost:8123
- PostgreSQL: localhost:5433

---

## 🔧 环境变量配置

### 后端环境变量

**文件位置**: `backend/.env`

```env
# 必需
GEMINI_API_KEY=your_gemini_api_key

# 可选
LANGSMITH_API_KEY=your_langsmith_api_key  # 用于追踪
QUERY_GENERATOR_MODEL=gemini-2.0-flash
REFLECTION_MODEL=gemini-2.5-flash
ANSWER_MODEL=gemini-2.5-pro
NUMBER_OF_INITIAL_QUERIES=3
MAX_RESEARCH_LOOPS=2
```

### 前端环境变量

**文件位置**: `frontend/.env`

```env
# Vite 自动处理
# 使用 import.meta.env.VITE_VARIABLE_NAME 访问
```

---

## 📊 部署架构图

```
┌─────────────────────────────────────────────────────────┐
│                     用户浏览器                           │
└─────────────────────┬───────────────────────────────────┘
                      │
                      │ HTTP/WebSocket
                      ▼
┌─────────────────────────────────────────────────────────┐
│              Nginx / Load Balancer                      │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│              LangGraph API 容器                         │
│  ┌───────────────────────────────────────────────────┐ │
│  │   FastAPI + LangGraph                            │ │
│  │                                                   │ │
│  │   ├── /app (StaticFiles: React 前端)            │ │
│  │   ├── /runs (LangGraph API)                     │ │
│  │   ├── /threads (LangGraph API)                  │ │
│  │   └── /docs (FastAPI Docs)                      │ │
│  └───────────────────────────────────────────────────┘ │
└────┬────────────────────────────────────────────┬───────┘
     │                                            │
     │ Pub-Sub                                    │ State
     ▼                                            ▼
┌──────────────────┐                   ┌──────────────────┐
│   Redis 容器     │                   │ PostgreSQL 容器  │
│  (消息代理)       │                   │  (状态存储)       │
└──────────────────┘                   └──────────────────┘
```

---

## ✅ 阶段 5 总结

### 关键收获

1. **FastAPI 集成**: 使用 StaticFiles 服务前端静态文件
2. **开发 vs 生产**: 理解两种环境的配置差异
3. **Docker 部署**: 多阶段构建,优化镜像大小
4. **Redis 作用**: Pub-Sub 消息代理,实时事件推送
5. **PostgreSQL 作用**: 状态持久化,任务队列管理

### 核心概念图

```
部署架构
    ├── 开发环境
    │   ├── 前端: Vite Dev Server (5173)
    │   ├── 后端: LangGraph Dev (2024)
    │   └── 数据: 内存存储
    │
    └── 生产环境
        ├── 前端: 静态文件 (FastAPI)
        ├── 后端: LangGraph API (8123)
        ├── Redis: Pub-Sub 消息代理
        └── PostgreSQL: 状态持久化
```

### 下一步

进入**阶段 6: 高级主题与扩展**,深入学习:
- LangGraph 高级特性
- Agent 设计模式
- 项目扩展方向

### 学习验证

完成本阶段后,确保能够:

- [ ] 理解开发环境和生产环境的区别
- [ ] 能解释为什么需要 Redis 和 PostgreSQL
- [ ] 成功构建和运行 Docker 镜像
- [ ] 理解 FastAPI 和 LangGraph API 的关系
- [ ] 能配置生产环境的环境变量

---

## 📚 延伸阅读

- [LangGraph Deployment Options](https://langchain-ai.github.io/langgraph/concepts/deployment_options/)
- [FastAPI Static Files](https://fastapi.tiangolo.com/tutorial/static-files/)
- [Docker Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Redis Pub-Sub](https://redis.io/docs/manual/pubsub/)
- [PostgreSQL JSONB](https://www.postgresql.org/docs/current/datatype-json.html)

---

*下一阶段: 深入学习高级主题与扩展*
