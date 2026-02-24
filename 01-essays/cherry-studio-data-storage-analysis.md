# Cherry Studio 数据存储分析

## 分析日期

2026-02-24

## 目录结构概览

CherryStudio 的数据存储位于 `C:\Users\29580\AppData\Roaming\CherryStudio\`

```text
CherryStudio/
├── .claude/                    # Claude Code 配置
├── blob_storage/              # Blob 存储
├── Cache/                     # 缓存
├── Code Cache/                # 代码缓存
├── Crashpad/                  # 崩溃报告
├── Data/                      # 主要数据目录 ⭐
│   ├── agents.db             # SQLite 助手数据库
│   ├── Files/                # 文件存储
│   ├── KnowledgeBase/        # 知识库
│   ├── Memory/               # 记忆存储
│   └── Notes/                # 笔记存储
├── IndexedDB/                 # IndexedDB 存储
├── Local Storage/             # Local Storage (leveldb) ⭐
├── Session Storage/           # 会话存储
├── Partitions/                # 分区数据
├── config.json                # 应用配置
├── Preferences                # 偏好设置
└── window-state.json          # 窗口状态
```

---

## 核心数据存储位置分析

### 1\. Local Storage (leveldb) - 助手库主要存储

**路径**: `Local Storage/leveldb/`

**文件列表**:

-   `000005.ldb` (158KB)
    
-   `001436.ldb` (158KB)
    
-   `001438.ldb` (178KB)
    
-   `001440.ldb` (178KB) - 最新
    

**内容分析**:

-   存储格式: LevelDB 键值对存储
    
-   发现数据片段: `{"assistants":"{\"defaultA>`
    
-   包含: `persist:cherry-studio` 命名空间
    
-   **这是自定义助手配置最可能的存储位置**
    

**键类型发现**:

-   `assistants` - 助手列表/配置
    
-   `assistant` - 单个助手数据
    
-   `agentSessionId_` - 助手会话ID
    

---

### 2\. Data/agents.db - SQLite 助手数据库

**路径**: `Data/agents.db`

**数据库结构**:

```sql
CREATE TABLE agents (
    id text PRIMARY KEY,
    type text NOT NULL,
    name text NOT NULL,
    description text,
    accessible_paths text,
    instructions text,
    model text NOT NULL,
    plan_model text,
    small_model text,
    mcps text,
    allowed_tools text,
    configuration text,
    created_at text NOT NULL,
    updated_at text NOT NULL
);

CREATE TABLE sessions (
    id text PRIMARY KEY,
    agent_type text NOT NULL,
    agent_id text NOT NULL,
    name text NOT NULL,
    description text,
    -- ... 其他字段
    slash_commands text
);

CREATE TABLE session_messages (
    id integer PRIMARY KEY AUTOINCREMENT,
    session_id text NOT NULL,
    role text NOT NULL,
    content text NOT NULL,
    metadata text,
    created_at text NOT NULL,
    updated_at text NOT NULL,
    agent_session_id text
);
```

**当前状态**: 数据库为空（无助手记录）

**用途**: 可能用于存储会话历史和消息，而非助手配置本身

---

### 3\. IndexedDB - 结构化数据存储

**路径**: `IndexedDB/file__0.indexeddb.leveldb/`

**文件**:

-   `000402.ldb` (1.5MB)
    
-   `000400.log` (951KB)
    

**内容**: 可能存储会话数据、消息历史

---

### 4\. Data/KnowledgeBase - 知识库向量数据库

**路径**: `Data/KnowledgeBase/l0ttBeEDPGs_LeqjwpOK9`

**格式**: SQLite 3.x 数据库

**结构**:

```sql
CREATE TABLE vectors (
    id TEXT PRIMARY KEY,
    pageContent TEXT UNIQUE,
    uniqueLoaderId TEXT NOT NULL,
    source TEXT NOT NULL,
    vector F32_BLOB,
    metadata TEXT
);
```

**用途**: 向量嵌入存储，用于知识库检索

---

### 5\. Data/Files/ - 文件存储

**路径**: `Data/Files/`

**文件**:

-   `custom-minapps.json` - 当前为空数组 `[]`
    

**用途**: 可能用于存储自定义迷你应用配置

---

## 辅助存储位置

### Partitions/webview/

WebView 进程的独立存储空间，包含：

-   `Local Storage/leveldb/`
    
-   `IndexedDB/`
    
-   `Session Storage/`
    
-   `DIPS` - SQLite 数据库（被锁定）
    

### Session Storage/

临时会话数据存储

### DIPS 文件

**路径**: `DIPS` 和 `DIPS-wal`

**格式**: SQLite 3.x 数据库

**状态**: 运行时被锁定

---

## 自定义助手数据提取方法

### 方法 1: 应用关闭后查看

```bash
# 关闭 Cherry Studio 后操作
sqlite3 Data/agents.db "SELECT * FROM agents;"
```

### 方法 2: LevelDB 读取工具

```bash
# 使用 Python plyvel 库
pip install plyvel
python -c "
import plyvel
db = plyvel.DB('Local Storage/leveldb')
for key, value in db:
    print(key.decode(), value.decode())
"
```

### 方法 3: 直接字符串提取

```bash
# 从 ldb 文件提取文本
strings "Local Storage/leveldb/"*.ldb | grep -i assistant
```

### 方法 4: 应用内导出

-   使用 Cherry Studio 内置的导出/备份功能
    
-   检查设置中的助手管理选项
    

---

## 配置文件

### config.json

```json
{
    "clientId": "0f59daa7-4ad1-4035-bc73-e0d97b015697",
    "theme": "system",
    "autoUpdate": true,
    "launchToTray": true,
    "gitBashPath": "D:\\Program Files\\Git\\bin\\bash.exe",
    "gitBashPathSource": "auto"
}
```

---

## 查找特定助手

### 查找名为"文件夹命名"的自定义助手

1.  **LevelDB 搜索**:
    
    ```bash
    grep -ra "文件夹命名" "Local Storage/leveldb/"
    ```
    
2.  **数据库查询** (应用关闭后):
    
    ```bash
    sqlite3 Data/agents.db "SELECT * FROM agents WHERE name LIKE '%文件夹%';"
    ```
    
3.  **十六进制查看**:
    
    ```bash
    hexdump -C "Local Storage/leveldb/"*.ldb | grep -i "文件夹"
    ```
    

---

## 数据格式推测

基于分析，助手数据可能以以下格式存储：

```json
{
  "assistants": {
    "defaultAssistant": "assistant_id",
    "customAssistants": [
      {
        "id": "uuid",
        "name": "文件夹命名",
        "description": "...",
        "instructions": "...",
        "model": "...",
        "configuration": {...}
      }
    ]
  }
}
```

---

## 总结

| 存储位置 | 用途  | 状态  |
| --- | --- | --- |
| `Local Storage/leveldb/` | 助手配置 | ⭐ 主要位置 |
| `Data/agents.db` | 会话/消息 | 空数据库 |
| `IndexedDB/` | 结构化数据 | 活跃使用 |
| `Data/KnowledgeBase/` | 向量存储 | 活跃使用 |
| `Data/Files/custom-minapps.json` | 迷你应用 | 空数组 |

---

## 后续分析建议

1.  在 Cherry Studio 关闭状态下重新分析数据库
    
2.  使用专门的 LevelDB 查看工具
    
3.  检查是否有备份文件
    
4.  查看应用源代码了解数据结构
    
5.  尝试创建新的助手并观察数据变化