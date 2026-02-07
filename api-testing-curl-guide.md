# API 测试与 Curl 命令指南 - 完整对话记录

> 本文档记录了关于 API 测试工具选择和 Curl 命令使用的完整对话过程。

---

## 对话 1：初始困惑

### 用户问题

我有一个困惑，你不需要阅读当前目录的文件，我只是想问你一个问题：

我是一位python后端开发程序员，我开发了web接口后，我想测试，目前的情况是：
生成接口的 curl 然后测试，### 我的回答

明白了。你目前用 curl 命令来测试 web 接口，这个困惑具体是指什么呢？比如：

1. **觉得 curl 测试不够方便**？（想找更高效的测试工具）
2. **想自动化测试流程**？（比如集成到 CI/CD 或编写测试脚本）
3. **想管理多个测试用例**？（比如保存常用请求、参数组合）
4. **想测试接口的各个方面**？（性能测试、并发测试、自动化回归测试）
5. **其他具体问题**？

或者你可以直接告诉我你的困惑是什么，这样我能更精准地帮你。

---

## 对话 2：详细说明困惑

### 用户问题

1. curl 虽然还算方便，但是文档中的 curl 没办法和接口同步（fastapi），接口变更后，curl 我还得同步改
2. 我也想自动化测试流程
3. 我确实想管理多个测试用例
虽然 postman 等工具可以，但是我用起来是在感觉不方便

### 我的回答

理解了！你的核心痛点是：

1. **文档与代码不同步**：接口改了，curl 命令要手动改
2. **需要自动化测试**：不想每次手动跑 curl
3. **用例管理麻烦**：Postman 这种 GUI 工具你觉得重且不方便

#### 针对FastAPI的推荐方案

基于你的技术栈（Python + FastAPI），我推荐以下方案：

##### 方案1：FastAPI TestClient + pytest（最推荐）

```python
# tests/test_api.py
from fastapi.testclient import TestClient
from main import app  # 你的 FastAPI app

client = TestClient(app)

def test_create_user():
    response = client.post("/users", json={
        "name": "张三",
        "email": "zhangsan@example.com"
    })
    assert response.status_code == 200
    assert response.json()["name"] == "张三"

def test_get_users():
    response = client.get("/users")
    assert response.status_code == 200
```

**优点**：
- 代码即文档，接口改了测试直接报错
- 完全自动化，可以集成到 CI/CD
- 可以用 pytest 的所有特性（fixture、parametrize、mark）

##### 方案2：httpx/requests + pytest

如果要测试真实 HTTP 请求（包括网络、中间件等）：

```python
# tests/test_api_real.py
import pytest
import httpx

BASE_URL = "http://localhost:8000"

@pytest.mark.parametrize("page,size", [(1, 10), (2, 20)])
def test_get_users_paginated(page, size):
    response = httpx.get(f"{BASE_URL}/users", params={
        "page": page,
        "size": size
    })
    assert response.status_code == 200
```

##### 方案3：VS Code REST Client（轻量级交互测试）

如果你偶尔需要手动测试，不想写代码：

```http
# tests/api.http
### 获取用户列表
GET http://localhost:8000/users?page=1&size=10

### 创建用户
POST http://localhost:8000/users
Content-Type: application/json

{
  "name": "张三",
  "email": "zhangsan@example.com"
}
```

安装 VS Code 插件 `rest-client`，点击 "Send Request" 即可测试。

#### 我的建议组合

**开发阶段**：
- 写 FastAPI 接口时，同步写 `TestClient` 测试
- 用 `pytest` 管理，可以用 `@pytest.mark.parametrize` 管理多个用例

**手动调试时**：
- 用 VS Code REST Client 或浏览器访问 `/docs`

**自动化测试流程**：
```bash
# 运行所有测试
pytest

# 运行特定标记的测试
pytest -m smoke

# 并发运行（加速）
pytest -n auto
```

---

## 对话 3：请求 Curl 示例

### 用户问题

你给我生成点 curl 例子

### 我的回答：基础 Curl 命令示例

好的，给你一些常用的 curl 示例：

#### 基础 HTTP 方法

```bash
# GET 请求 - 获取数据
curl -X GET "http://localhost:8000/users"

# GET 带查询参数
curl -X GET "http://localhost:8000/users?page=1&size=10&status=active"

# POST - 创建数据（JSON）
curl -X POST "http://localhost:8000/users" \
  -H "Content-Type: application/json" \
  -d '{"name": "张三", "email": "zhangsan@example.com", "age": 25}'

# PUT - 更新数据
curl -X PUT "http://localhost:8000/users/123" \
  -H "Content-Type: application/json" \
  -d '{"name": "李四", "email": "lisi@example.com"}'

# PATCH - 部分更新
curl -X PATCH "http://localhost:8000/users/123" \
  -H "Content-Type: application/json" \
  -d '{"age": 26}'

# DELETE - 删除数据
curl -X DELETE "http://localhost:8000/users/123"
```

#### 认证相关

```bash
# Bearer Token（JWT）
curl -X GET "http://localhost:8000/protected" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# Basic Auth
curl -X GET "http://localhost:8000/protected" \
  -u "username:password"

# API Key
curl -X GET "http://localhost:8000/data" \
  -H "X-API-Key: your-api-key-here"
```

#### 文件上传

```bash
# 单文件上传
curl -X POST "http://localhost:8000/upload" \
  -F "file=@/path/to/file.jpg" \
  -F "description=这是一张图片"

# 多文件上传
curl -X POST "http://localhost:8000/upload/multiple" \
  -F "files=@/path/to/file1.jpg" \
  -F "files=@/path/to/file2.pdf"

# 上传并带其他 JSON 字段
curl -X POST "http://localhost:8000/upload" \
  -F "file=@document.pdf" \
  -F 'metadata={"title": "报告", "author": "张三"}'
```

#### 表单提交

```bash
# application/x-www-form-urlencoded
curl -X POST "http://localhost:8000/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=123456&remember_me=true"

# 或者用 --data-urlencode（自动编码）
curl -X POST "http://localhost:8000/search" \
  --data-urlencode "keyword=Python 编程" \
  --data-urlencode "page=1"
```

#### 实用调试参数

```bash
# 查看完整响应头
curl -X GET "http://localhost:8000/users" \
  -i

# 只查看响应头
curl -X GET "http://localhost:8000/users" \
  -I

# 显示详细请求过程（调试用）
curl -X GET "http://localhost:8000/users" \
  -v

# 格式化输出 JSON（需要 jq）
curl -X GET "http://localhost:8000/users" | jq .

# 保存响应到文件
curl -X GET "http://localhost:8000/users" \
  -o response.json

# 从文件读取请求体
curl -X POST "http://localhost:8000/users" \
  -H "Content-Type: application/json" \
  -d @request_body.json
```

#### 复杂场景

```bash
# 带自定义 Headers
curl -X GET "http://localhost:8000/data" \
  -H "Accept: application/json" \
  -H "User-Language: zh-CN" \
  -H "X-Request-ID: abc123"

# 发送 Cookie
curl -X GET "http://localhost:8000/profile" \
  -H "Cookie: session_id=xyz789; user_pref=dark_mode"

# 保存 Cookie 到文件（后续请求复用）
curl -X POST "http://localhost:8000/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "123456"}' \
  -c cookies.txt

curl -X GET "http://localhost:8000/protected" \
  -b cookies.txt

# 设置超时
curl -X GET "http://localhost:8000/slow-api" \
  --max-time 30 \
  --connect-timeout 10

# 忽略 SSL 证书（仅测试用）
curl -X GET "https://localhost:8000/users" \
  -k
```

#### 实际业务场景示例

```bash
# 用户注册
curl -X POST "http://localhost:8000/api/v1/auth/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "zhangsan",
    "password": "P@ssw0rd123",
    "email": "zhangsan@example.com",
    "phone": "13800138000"
  }'

# 获取商品列表（带筛选和分页）
curl -X GET "http://localhost:8000/api/v1/products" \
  -G \
  --data-urlencode "category=电子产品" \
  --data-urlencode "min_price=1000" \
  --data-urlencode "max_price=5000" \
  --data-urlencode "sort=price_desc" \
  --data-urlencode "page=2" \
  --data-urlencode "page_size=20"

# 创建订单
curl -X POST "http://localhost:8000/api/v1/orders" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your-jwt-token" \
  -d '{
    "items": [
      {"product_id": 101, "quantity": 2},
      {"product_id": 205, "quantity": 1}
    ],
    "shipping_address": {
      "province": "北京市",
      "city": "朝阳区",
      "detail": "xxx街道xxx号",
      "postcode": "100000"
    }
  }'
```

#### Windows 注意事项

如果在 Windows CMD 中使用：
- 双引号内的 JSON 需要转义双引号：`'{\"name\": \"张三\"}'`
- 或者使用 PowerShell，单引号内不需要转义

PowerShell 示例：
```powershell
curl -X POST "http://localhost:8000/users" `
  -H "Content-Type: application/json" `
  -d '{"name": "张三", "email": "zhangsan@example.com"}'
```

---

## 对话 4：请求 Httpbin 示例

### 用户问题

来点调用 httpbin 的 curl

### 我的回答：Httpbin 的 Curl 测试示例

好的，httpbin 是个很棒的测试工具！给你一些常用示例：

#### 基础请求

```bash
# GET 请求 - 返回请求信息
curl "https://httpbin.org/get"

# 返回客户端 IP 信息
curl "https://httpbin.org/ip"

# 返回请求头信息
curl "https://httpbin.org/headers"

# 返回所有请求信息（包含 headers、origin 等）
curl "https://httpbin.org/anything"

# 延迟响应（模拟慢接口，3秒）
curl "https://httpbin.org/delay/3"

# 返回状态码 200（默认）
curl "https://httpbin.org/status/200"

# 返回各种状态码
curl "https://httpbin.org/status/404"
curl "https://httpbin.org/status/500"
curl "https://httpbin.org/status/403"
curl "https://httpbin.org/status/401"
```

#### POST 请求

```bash
# 发送 JSON 数据
curl -X POST "https://httpbin.org/post" \
  -H "Content-Type: application/json" \
  -d '{"name": "张三", "age": 25, "city": "北京"}'

# 发送表单数据
curl -X POST "https://httpbin.org/post" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=admin&password=123456&remember=true"

# 发送纯文本
curl -X POST "https://httpbin.org/post" \
  -H "Content-Type: text/plain" \
  -d "这是纯文本内容"

# 发送 JSON 并格式化输出
curl -X POST "https://httpbin.org/post" \
  -H "Content-Type: application/json" \
  -d '{"foo": "bar"}' | jq .
```

#### 认证测试

```bash
# Basic Auth - 成功
curl -X GET "https://httpbin.org/basic-auth/user/passwd" \
  -u "user:passwd"

# Basic Auth - 失败
curl -X GET "https://httpbin.org/basic-auth/user/passwd"

# Bearer Token
curl -X GET "https://httpbin.org/bearer" \
  -H "Authorization: Bearer your-token-here"

# 摘要认证
curl -X GET "https://httpbin.org/digest-auth/auth/user/passwd" \
  -u "user:passwd"

# 隐藏的 Basic Auth（401 响应）
curl "https://httpbin.org/hidden-basic-auth/user/passwd"
```

#### Header 和 Cookie 测试

```bash
# 设置自定义 Header
curl "https://httpbin.org/headers" \
  -H "X-Custom-Header: MyValue" \
  -H "User-Agent: MyCustomAgent/1.0"

# 返回请求的 User-Agent
curl "https://httpbin.org/user-agent"

# 设置 Cookie
curl "https://httpbin.org/cookies" \
  -H "Cookie: name1=value1; name2=value2"

# 设置 Cookie（另一种方式）
curl "https://httpbin.org/cookies/set?name=张三&city=北京"

# 删除 Cookie
curl "https://httpbin.org/cookies/delete?session"
```

#### 重定向测试

```bash
# 重定向到指定 URL（最多 3 次）
curl "https://httpbin.org/redirect/3"

# 重定向到指定 URL
curl "https://httpbin.org/redirect-to?url=https://www.baidu.com"

# 绝对重定向（302）
curl "https://httpbin.org/relative-redirect/3"

# 不跟随重定向（查看响应头）
curl -I "https://httpbin.org/redirect/1"

# 获取重定向链
curl -L "https://httpbin.org/redirect/5"
```

#### 参数和编码测试

```bash
# 返回查询参数
curl "https://httpbin.org/get?name=张三&age=25&city=北京"

# URL 编码测试
curl "https://httpbin.org/encoding/utf8"

# Gzip 压缩响应
curl "https://httpbin.org/gzip" \
  -H "Accept-Encoding: gzip"

# Deflate 压缩响应
curl "https://httpbin.org/deflate" \
  -H "Accept-Encoding: deflate"

# 返回响应头
curl -I "https://httpbin.org/response-headers?Content-Type=text/plain&X-Custom-Header=MyValue"
```

#### 文件上传

```bash
# 上传单个文件
curl "https://httpbin.org/post" \
  -F "file=@test.txt" \
  -F "description=测试文件"

# 上传多个文件
curl "https://httpbin.org/post" \
  -F "files=@file1.txt" \
  -F "files=@file2.jpg"

# 上传文件并带其他字段
curl "https://httpbin.org/post" \
  -F "upload=@document.pdf" \
  -F "title=测试文档" \
  -F "author=张三"

# 只上传文件（multipart/form-data）
curl "https://httpbin.org/post" \
  -F "file=@/path/to/file.jpg;type=image/jpeg"
```

#### 流式和大数据测试

```bash
# 流式响应（每秒 1 行，共 10 行）
curl "https://httpbin.org/stream/5"

# 流式 JSON（每秒 1 个对象）
curl "https://httpbin.org/stream/10"

# 返回指定字节的随机数据
curl "https://httpbin.org/bytes/1024"

# 返回指定长度的随机数据
curl "https://httpbin.org/drip?duration=1&numbytes=10&code=200"

# 网络延迟测试（1-5 秒随机延迟）
curl "https://httpbin.org/delay/2"
```

#### 方法测试

```bash
# 各种 HTTP 方法
curl -X GET "https://httpbin.org/get"
curl -X POST "https://httpbin.org/post" \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
curl -X PUT "https://httpbin.org/put" \
  -H "Content-Type: application/json" \
  -d '{"updated": "data"}'
curl -X PATCH "https://httpbin.org/patch" \
  -H "Content-Type: application/json" \
  -d '{"patch": "data"}'
curl -X DELETE "https://httpbin.org/delete"

# 返回请求的方法
curl -X POST "https://httpbin.org/method"
```

#### 时间测试

```bash
# 返回当前时间
curl "https://httpbin.org/time"

# 返回当前时间戳
curl "https://httpbin.org/epoch"

# 延迟后返回时间（2 秒）
curl "https://httpbin.org/delay/2"
```

#### 实用调试组合

```bash
# 查看完整请求和响应（包含 header）
curl -v "https://httpbin.org/get?foo=bar"

# 只看响应头
curl -I "https://httpbin.org/get"

# 查看响应时间
curl -w "\nTotal time: %{time_total}s\n" "https://httpbin.org/delay/1"

# 保存响应到文件
curl "https://httpbin.org/get" -o response.json

# 测试连接超时
curl --connect-timeout 1 "https://httpbin.org/delay/5"

# 测试总超时
curl --max-time 2 "https://httpbin.org/delay/5"
```

#### 实际业务场景模拟

```bash
# 模拟 API 调用（带认证和自定义头）
curl "https://httpbin.org/bearer" \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "X-Request-ID: abc123" \
  -H "X-Client-Version: 1.0.0"

# 模拟分页请求
curl "https://httpbin.org/get?page=1&page_size=10&sort=created_at"

# 模拟过滤请求
curl "https://httpbin.org/get?status=active&category=electronics&min_price=100"

# 模拟搜索请求
curl "https://httpbin.org/get?keyword=Python编程&page=1&limit=20"
```

#### WebSocket 和其他协议测试

```bash
# 返回 WebSocket 地址（需要专门的 WebSocket 客户端）
curl "https://httpbin.org/websocket"

# HTML 表单（浏览器可访问）
curl "https://httpbin.org/forms/post"

# 返回 XML
curl "https://httpbin.org/xml"

# 返回 HTML
curl "https://httpbin.org/html"

# 返回图片（PNG）
curl "https://httpbin.org/image" -o image.png

# 返回图片（指定格式）
curl "https://httpbin.org/image/jpeg" -o image.jpg
curl "https://httpbin.org/image/png" -o image.png
curl "https://httpbin.org/image/svg" -o image.svg
```

#### 高级测试场景

```bash
# 测试缓存控制
curl -I "https://httpbin.org/cache" \
  -H "If-None-Match: my-etag"

# 返回所有环境变量
curl "https://httpbin.org/anything/env"

# 返回请求的 IP 信息
curl "https://httpbin.org/ip"

# 返回机器人检测信息
curl "https://httpbin.org/robots.txt"

# 拒绝请求（403 Forbidden）
curl "https://httpbin.org/deny"

# 返回 UUID
curl "https://httpbin.org/uuid"
```

#### 压力测试（慎用）

```bash
# 连续请求（模拟并发）
for i in {1..10}; do curl "https://httpbin.org/get" & done

# 带计时的请求
curl -w "@-" "https://httpbin.org/get" <<< 'time_namelookup:  %{time_namelookup}\ntime_connect:     %{time_connect}\ntime_appconnect:  %{time_appconnect}\ntime_pretransfer: %{time_pretransfer}\ntime_starttransfer: %{time_starttransfer}\ntime_total:       %{time_total}\n'
```

---

## 对话 5：保存文档

### 用户问题

把你当前的所有回答原封不动地写入到 D:\Users\29580\TextNotes 中，创建一个新文件就行

### 用户问题（补充）

再把问题也加上

### 我的操作

已将完整的对话记录（包括问题和回答）保存到文档中。

---

## 总结

本文档记录了关于 Python 后端 API 测试的完整对话，包含：

1. **问题识别**：curl 测试的痛点分析
   - 文档与代码不同步
   - 需要自动化测试
   - 测试用例管理困难

2. **解决方案推荐**：针对 FastAPI 的三种测试方案
   - FastAPI TestClient + pytest（最推荐）
   - httpx/requests + pytest（真实 HTTP 测试）
   - VS Code REST Client（轻量级手动测试）

3. **实用命令示例**：
   - 基础 curl 命令（GET、POST、PUT、PATCH、DELETE）
   - 认证测试（Bearer Token、Basic Auth、API Key）
   - 文件上传、表单提交、Cookie 管理
   - 调试参数和复杂场景

4. **httpbin.org 测试用例**：
   - 完整的 HTTP 方法测试
   - 认证、重定向、编码测试
   - 流式响应、时间测试、压力测试
   - 实际业务场景模拟

这些示例可以直接复制使用，涵盖了日常开发中的大部分 API 测试场景。

---

*文档创建时间：2026-02-07*
*记录工具：Claude Code*
