# bru脚本编写规则总结

根据您提供的多个文档链接（主要来自 [https://docs.usebruno.com](https://docs.usebruno.com)），结合知识库中的内容（特别是 `samples` 和 `tag-reference` 文件），我为您整理了一份 **.bru 脚本编写规则总结**。这份总结涵盖了 `.bru` 文件的核心语法、结构、常用标签和最佳实践。

---

## 📄 什么是 .bru 文件？

`.bru` 是 Bruno 使用的**纯文本、人类可读的请求定义文件格式**，基于一种称为 **Bru Markup Language (Bru Lang)** 的 DSL（领域特定语言）。它用于描述 HTTP 请求、GraphQL 查询、gRPC 调用等，并支持测试脚本、变量和文档。

---

## 🔤 核心语法规则

### 1. **基本结构：标签 + 块**
每个 `.bru` 文件由一系列 **标签（tags）** 组成，每个标签后跟一个用大括号 `{}` 包裹的**值块**。

```bru
<标签> {
  <值>
}
```

- **标签名**：不区分大小写，但通常使用小写（如 `get`, `post`, `url`）。
- **值块**：可以是单行或多行。多行时，每行内容直接写在大括号内。

### 2. **HTTP 方法标签**
定义请求的 HTTP 方法。必须且只能有一个。

| 标签 | 说明 |
| :--- | :--- |
| `get` | GET 请求 |
| `post` | POST 请求 |
| `put` | PUT 请求 |
| `patch` | PATCH 请求 |
| `delete` | DELETE 请求 |
| `head` | HEAD 请求 |
| `options` | OPTIONS 请求 |
| `trace` | TRACE 请求 |
| `connect` | CONNECT 请求 |

### 3. **核心请求配置标签**

| 标签 | 说明 | 示例 |
| :--- | :--- | :--- |
| `url` | **必需**。请求的完整 URL 或带变量的 URL。 | `url: https://api.example.com/users/{{userId}}` |
| `query` | 定义 URL 查询参数。 | `query { page=1 limit=10 }` |
| `headers` | 定义请求头。 | `headers { Content-Type: application/json Authorization: Bearer {{token}} }` |
| `body` | 定义请求体。支持 JSON、表单等多种格式。 | `body { { "name": "Alice" } }` |
| `auth` | （可选）覆盖集合或工作区的认证设置。 | `auth { type: bearer token: {{myToken}} }` |

### 4. **脚本与测试标签**

| 标签 | 说明 |
| :--- | :--- |
| `pre-request` | 在发送请求前执行的 JavaScript 脚本。 |
| `post-response` | 在收到响应后执行的 JavaScript 脚本，常用于断言（测试）。 |
| `vars` | 定义此请求级别的变量。 |

### 5. **文档标签**

| 标签 | 说明 |
| :--- | :--- |
| `docs` | 为当前请求添加 Markdown 格式的文档。 |

---

## 🧪 变量与插值

Bruno 支持强大的变量系统，可以在 `.bru` 文件的任何地方使用 `{{variableName}}` 语法进行插值。

- **变量来源**：环境变量、集合变量、文件夹变量、全局变量、脚本中动态设置的变量等。
- **示例**：
  ```bru
  url: https://{{host}}/api/v1/users
  headers {
    Authorization: Bearer {{authToken}}
  }
  ```

---

## 📝 编写示例

### 示例 1: 基础 GET 请求
```bru
get {
  url: https://api.github.com/users/usebruno
}
```

### 示例 2: 带 Header 和 Query 的 GET 请求
```bru
get {
  url: https://api.textlocal.in/send
}
query {
  apiKey=secret
  numbers=9988776655
  message=hello
}
headers {
  content-type: application/json
  Authorization: Bearer topsecret
}
```

### 示例 3: 带 JSON Body 的 POST 请求
```bru
post {
  url: https://api.textlocal.in/send
}
body {
  {
    "apiKey": "secret",
    "numbers": "9988776655",
    "message": "Woof! lets play with some apis"
  }
}
headers {
  content-type: application/json
  Authorization: Bearer topsecret
}
```

### 示例 4: 带测试脚本的请求
```bru
get {
  url: https://jsonplaceholder.typicode.com/posts/1
}
post-response {
  // 断言状态码为200
  bru.assert(response.status === 200);
  // 断言响应体包含特定ID
  bru.assert(response.body.id === 1);
}
```

---

## ⚙️ 最佳实践

1. **模块化**：将相关的请求组织到**文件夹（Folder）** 中，共享文件夹级别的变量和认证。
2. **使用变量**：避免硬编码 URL、密钥等敏感或可变信息，使用 `{{}}` 插值。
3. **编写测试**：为每个关键请求添加 `post-response` 脚本，确保 API 行为符合预期。
4. **添加文档**：利用 `docs` 标签为复杂的请求提供上下文说明，方便团队协作。
5. **版本控制**：`.bru` 文件是纯文本，非常适合与 Git 等版本控制系统集成。

---

## 📚 总结

`.bru` 文件是一种简洁、强大且易于维护的 API 请求定义方式。其核心在于**标签化的声明式语法**，配合**变量插值**和**JavaScript 脚本**，能够满足从简单调用到复杂自动化测试的各种需求。

通过遵循以上规则和最佳实践，您可以高效地在 Bruno 中管理和运行您的 API 工作流。