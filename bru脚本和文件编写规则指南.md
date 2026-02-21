# bru脚本和文件编写规则指南

根据提供的 Bruno 官方文档链接和 `.bru` 标记语言的特性，我为您总结了一份详尽的 **`.bru` 脚本和文件编写规则指南**。

`.bru` 是 Bruno API 客户端专用的纯文本标记语言。它的设计初衷是**对 Git 版本控制极为友好**，易于人类阅读、编写和进行代码审查（Code Review）。

以下是 `.bru` 脚本的核心编写规则与语法总结：

### 1. 文件基本结构与语法特征

* **隐式键值对与块结构**：顶层通常是隐式的键值结构（类似 Multimap），不需要严格的外层大括号。
* **数据类型**：支持字符串（String）、数字（Number）、布尔值（Boolean）和空值（Null）。
* **缩进与空白**：基于结构化的换行和缩进，空白行默认会被忽略，不需要每行末尾加逗号。

### 2. 核心标签块 (Tag Reference)

一个标准的 `.bru` 文件由多个“块（Block）”组成，常用的块及其规则如下：

* **`meta` (元数据块)**：
用于存储请求的基础信息。
```bru
meta {
  name: 获取用户列表
  type: http
  seq: 1
  tags: [ smoke, sanity ]
}

```


*(注：`seq` 决定 UI 里的排序，`type` 一般是 http 或 graphql)*
* **HTTP 方法块 (`get`, `post`, `put`, `delete` 等)**：
定义请求的 URL 及其主要配置（如 body 类型和 auth 类型）。
```bru
post {
  url: https://api.example.com/users
  body: json
  auth: bearer
}

```


* **参数与请求头块 (`query`, `headers`)**：
直接以键值对形式编写。
```bru
query {
  page: 1
  limit: 20
}
headers {
  Content-Type: application/json
}

```


* **请求体块 (`body:*`)**：
支持多种数据格式，如 `body:json`、`body:text`、`body:xml`、`body:form-urlencoded` 等。如果声明为 JSON，内部可以直接写原生的 JSON 代码。
```bru
body:json {
  {
    "id": 1,
    "name": "Bruno"
  }
}

```


* **鉴权块 (`auth:*`)**：
例如声明了 `auth:bearer` 后，需要提供对应的 token 配置。支持 `inherit`（继承父级文件夹配置）或使用变量如 `{{token}}`。

### 3. 断言规则 (Assertions)

Bruno 提供了声明式的 `assert` 块，让你无需写 JS 代码即可完成测试：

* **语法格式**：`表达式: 运算符 预期值`
* **常用示例**：
```bru
assert {
  res.status: eq 200                    // 状态码等于 200
  res.body.status: equals success       // 响应体中的 status 字段为 success
  res.body.message: contains created    // 包含特定文本
  res.body.users: isNotEmpty            // 数组或对象非空
  res.body.users[0].name: eq Alice      // 支持 JSON 嵌套或数组索引取值
}

```



### 4. 变量流转块 (Variables)

你可以声明式地在请求前后提取并保存变量，而不需要写 JavaScript。

* **`vars:pre-request`**：请求前处理变量。
* **`vars:post-response`**：请求后从响应体提取变量。
```bru
vars:post-response {
  token: res.body.access_token   // 直接将响应体中的 access_token 赋值给变量 token
  userId: res.body.data.id
}

```



### 5. JavaScript 脚本编写规则 (Scripting)

如果声明式规则不满足需求，Bruno 允许你使用原生 JavaScript 编写前后置脚本和测试用例：

* **块定义**：使用 `script:pre-request` (前置脚本)、`script:post-response` (后置脚本) 或 `tests` (测试脚本)。
* **内置对象**：
* **`req` (Request Object)**：可读取或修改即将发送的请求。例如 `req.setBody(data)`，`req.setHeader('X-Custom', 'Value')`。
* **`res` (Response Object)**：获取响应结果。例如 `res.getStatus()`，`res.getBody()`。
* **`bru` (Bruno Context Object)**：核心对象，用于操作变量链。例如 `bru.setVar("key", "value")`、`bru.getEnvVar("key")`。


* **引入外部库**：
你可以直接 `require()` 引入 Node.js 内置库，甚至引入当前文件夹下的其它外部 `.js` 脚本文件：
```bru
script:pre-request {
  const moment = require('moment');
  bru.setVar("currentDate", moment().format("YYYY-MM-DD"));

  // 也可以引入相对路径自己的公共逻辑
  const myUtils = require('./utils.js');
}

```



### 总结范例

一个典型的、完整的 `.bru` 脚本大致长这样：

```bru
meta {
  name: 创建新订单
  type: http
  seq: 2
}

post {
  url: {{baseUrl}}/api/orders
  body: json
  auth: bearer
}

headers {
  X-Request-Id: {{reqId}}
}

auth:bearer {
  token: {{accessToken}}
}

body:json {
  {
    "itemId": 99,
    "quantity": 1
  }
}

script:pre-request {
  // 生成动态 Request ID 并设为临时变量
  const uuid = require('uuid');
  bru.setVar("reqId", uuid.v4());
}

vars:post-response {
  // 提取订单 ID 供后续请求复用 (Request Chaining)
  orderId: res.body.data.orderId
}

assert {
  res.status: eq 201
  res.body.success: eq true
}

```