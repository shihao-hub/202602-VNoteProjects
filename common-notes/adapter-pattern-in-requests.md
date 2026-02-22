# Requests 库中的适配器模式与 Mock 对象

## 背景

在阅读 `requests` 库源码时，发现了一个有趣的设计：为了复用 Python 标准库 `http.cookiejar` 的功能，使用了 Mock 对象来适配接口。

## 核心问题

```
http.cookiejar.CookieJar.extract_cookies(response, request)
                    │
                    │ 期望接收标准库 urllib2.Request 的接口
                    ▼
        ┌─────────────────────────────┐
        │   urllib2.Request 接口      │
        ├─────────────────────────────┤
        │ get_type()                  │
        │ get_host()                  │
        │ get_full_url()              │
        │ is_unverifiable()           │
        │ get_header(name)            │
        └─────────────────────────────┘
```

但 `requests` 库使用的是 `urllib3`，它的 `Request` 对象接口完全不同：

```python
# requests/urllib3 的 Request
class Request:
    method = "GET"
    url = "https://example.com"
    headers = {"User-Agent": "..."}

    # ❌ 没有 get_type()、get_host() 等方法
```

## 适配器模式解决方案

### 核心思想

> 适配器模式（Adapter Pattern）：将一个类的接口转换成客户希望的另一个接口，使得原本由于接口不兼容而不能一起工作的那些类可以一起工作。

### MockRequest 实现

```python
# 文件位置: requests/cookies.py:23-101

class MockRequest:
    """将 requests.Request 包装成 urllib2.Request 的样子"""

    def __init__(self, request):
        self._r = request           # 持有原始 request
        self._new_headers = {}      # 存储新生成的 headers（如 Cookie）
        self.type = urlparse(self._r.url).scheme  # http / https

    # 以下是 urllib2.Request 的接口
    def get_type(self):
        return self.type

    def get_host(self):
        return urlparse(self._r.url).netloc

    def get_full_url(self):
        if not self._r.headers.get("Host"):
            return self._r.url
        # 如果设置了 Host 头，需要重构 URL
        host = self._r.headers["Host"]
        parsed = urlparse(self._r.url)
        return urlunparse([parsed.scheme, host, parsed.path,
                          parsed.params, parsed.query, parsed.fragment])

    def is_unverifiable(self):
        return True

    def get_header(self, name, default=None):
        return self._r.headers.get(name, self._new_headers.get(name, default))
```

### MockResponse 实现

```python
# 文件位置: requests/cookies.py:103-120+

class MockResponse:
    """包装 httplib.HTTPMessage，让 cookiejar 能读取响应头"""

    def __init__(self, headers):
        self._headers = headers  # HTTP 响应头（包含 Set-Cookie）

    def info(self):
        return self._headers  # cookiejar 通过 info() 获取头
```

### 使用方式

```python
# 文件位置: requests/cookies.py:124-137

def extract_cookies_to_jar(jar, request, response):
    """从响应中提取 cookies 并保存到 CookieJar"""
    if not (hasattr(response, "_original_response") and response._original_response):
        return

    # 创建 Mock 对象适配标准库接口
    req = MockRequest(request)
    res = MockResponse(response._original_response.msg)

    # 调用标准库方法提取 cookies
    jar.extract_cookies(res, req)
```

## 完整流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                      HTTP Response                              │
│                Set-Cookie: session=abc123                       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   urllib3.HTTPResponse                          │
│              ._original_response.msg (HTTPMessage)               │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     MockResponse                                 │
│                   .info() → HTTPMessage                          │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│           jar.extract_cookies(res, req)                         │
│           ← Python 标准库 http.cookiejar                         │
│           (解析 Set-Cookie，根据 domain/path 决定是否保存)       │
└─────────────────────────────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CookieJar                                     │
│           保存了 Cookie(name=session, value=abc123)              │
└─────────────────────────────────────────────────────────────────┘
```

## 为什么叫 "Mock"？

这里用 "Mock" 这个名字其实容易引起误解。更准确的叫法应该是 **Adapter（适配器）** 或 **Wrapper（包装器）**：

| 特征 | Mock 对象 | 适配器 |
|------|-----------|--------|
| 目的 | 隔离依赖进行测试 | 连接不兼容的接口 |
| 实现 | 通常返回假数据 | 转换调用真实对象 |
| 使用场景 | 单元测试 | 生产代码 |

`MockRequest/MockResponse` 虽然名字叫 Mock，但本质是 **Adapter**，用于连接新旧接口。

## 适配器模式图示

```
┌──────────────────────────────────────────────────────────────┐
│                    旧代码期望的接口                            │
│           (http.cookiejar，期望 urllib2.Request)              │
└──────────────────────────────────────────────────────────────┘
                           │
                           │ 接口不兼容
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                      新的实现                                 │
│              (requests.Request，用 urllib3)                   │
└──────────────────────────────────────────────────────────────┘
                           │
                           │ 需要连接
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                    MockRequest 适配器                          │
│  实现旧接口，内部调用新实现                                    │
└──────────────────────────────────────────────────────────────┘
```

## 接口对照表

| urllib2.Request 期望 | MockRequest 实现 | 数据来源 |
|---------------------|------------------|----------|
| `get_type()` | `return self.type` | `urlparse(request.url).scheme` |
| `get_host()` | `return urlparse(self._r.url).netloc` | `request.url` |
| `get_full_url()` | 返回完整 URL（处理 Host 头） | `request.url` + `request.headers` |
| `is_unverifiable()` | `return True` | 固定值 |
| `get_header(name)` | 从 headers 读取 | `request.headers` |

## 总结

| 问题 | 答案 |
|------|------|
| 为什么要适配？ | `requests.Request` 不符合标准库 `cookiejar` 期望的接口 |
| 用了什么模式？ | **适配器模式**（Adapter Pattern） |
| 为什么叫 Mock？ | 历史命名习惯，本质是 Adapter/Wrapper |
| 有什么好处？ | 复用标准库的 cookie 处理逻辑，无需重新实现 |

> 简单说：**旧代码（cookiejar）只认旧接口，新代码（requests）用新结构，MockRequest 就是翻译官，让旧代码能用新数据。**

## 参考源码

- `requests/cookies.py:23-101` - MockRequest 类
- `requests/cookies.py:103-120+` - MockResponse 类
- `requests/cookies.py:124-137` - extract_cookies_to_jar 函数
- `requests/sessions.py:718-725` - Cookie 持久化调用
