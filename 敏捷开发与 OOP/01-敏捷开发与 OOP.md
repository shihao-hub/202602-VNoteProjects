# 敏捷开发与 OOP 能力提升

## 第一部分：如何思考将功能抽象为独立类

### 设计思维分析

#### 1\. 单一职责原则 (SRP)

**糟糕的设计 - 一个类承担太多职责**：

```python
class UrlCrawler:
    def __init__(self, start_url, url_prefix=None, max_pages=500, timeout=10, user_agent="..."):
        self.start_url = start_url
        self.url_prefix = url_prefix
        self.max_pages = max_pages
        self.timeout = timeout
        self.user_agent = user_agent
        # ... 还有爬取逻辑、链接提取、结果输出等
```

**思考过程**：

-   这个类既要管理配置，又要负责爬取，还要处理输出
    
-   如果配置格式变了？需要修改 `UrlCrawler`
    
-   如果要从 requests 换成 Playwright？需要修改 `UrlCrawler`
    
-   一个类有多个"改变的理由" → 违反 SRP
    

**抽象方向**：把配置管理分离出来

```python
@dataclass
class CrawlerConfig:
    """只负责配置的存储和验证"""
    start_url: str
    url_prefix: str | None = None
    max_pages: int = 500
```

#### 2\. 依赖倒置原则 (DIP) - self.fetcher 的思考

**场景演化**：

```python
# 第一版 - 直接用 requests
class UrlCrawler:
    def crawl(self):
        response = requests.get(url)  # 硬编码依赖
```

**思考**：

-   如果要支持 JavaScript 渲染的页面怎么办？
    
-   如果要支持代理池、重试策略怎么办？
    
-   如果要 mock 测试怎么办？
    

**抽象方向**：定义抽象接口，依赖抽象而非具体实现

```python
# 定义"契约"
class WebFetcher(ABC):
    @abstractmethod
    def fetch(self, url: str) -> str:
        pass

# 具体实现可以替换
class RequestsFetcher(WebFetcher): ...
class PlaywrightFetcher(WebFetcher): ...

# UrlCrawler 只依赖抽象
class UrlCrawler:
    def __init__(self, config: CrawlerConfig, fetcher: WebFetcher):
        self.fetcher = fetcher  # 不关心具体是哪种 fetcher
```

#### 3\. 开闭原则 (OCP) - 对扩展开放，对修改关闭

```python
# 添加新功能时，不应该修改现有代码
# 只需要添加新的 Fetcher 实现
class SeleniumFetcher(WebFetcher):
    """新增 Selenium 支持，无需改动 UrlCrawler"""
    def fetch(self, url: str) -> str:
        driver.get(url)
        return driver.page_source
```

#### 4\. 可测试性 - 分离后容易测试

```python
# 分离前 - 难以测试
crawler = UrlCrawler(start_url="...")
crawler.crawl()  # 真的会发起网络请求，测试慢且不稳定

# 分离后 - 可以注入 mock
class MockFetcher(WebFetcher):
    def fetch(self, url: str) -> str:
        return "<html><a href='/test'>link</a></html>"

crawler = UrlCrawler(config, MockFetcher())  # 测试快速、可靠
```

#### 5\. 思考问题的自检清单

当你写代码时，问自己这些问题：

| 问题  | 如果答案是"是" → 考虑分离 |
| --- | --- |
| 这个类是否需要理解 HTTP 请求细节？ | 分离成 `Fetcher` |
| 配置参数是否超过 3 个？ | 分离成 `Config` 类 |
| 是否需要支持多种实现方式？ | 定义抽象基类 |
| 单元测试时是否需要 mock 网络请求？ | 注入依赖 |
| 更换底层库时需要修改核心逻辑？ | 依赖抽象 |

#### 实际重构路径

```text
第一阶段: 全部在 UrlCrawler 中
    ↓
第二阶段: 分离 Config 类
    ↓
第三阶段: 提取 Fetcher 接口
    ↓
第四阶段: 完整的组件化设计
```

---

## 第二部分：在敏捷开发中提升 OOP 能力

### 一、日常编码习惯 - 防御面条代码

#### 1\. 写完第一版后，立即进行"职责检查"

```python
# 第一版 - 快速实现（面条式）
def crawl_urls(start_url):
    urls = set()
    queue = [start_url]
    visited = set()
    max_pages = 500
    timeout = 10
    user_agent = "Mozilla/5.0..."

    while queue and len(visited) < max_pages:
        url = queue.pop(0)
        if url in visited:
            continue
        response = requests.get(url, headers={"User-Agent": user_agent}, timeout=timeout)
        # ... 50 行爬取逻辑
        # ... 30 行解析逻辑
        # ... 20 行输出逻辑

    return urls
```

**检查清单**（每个函数/方法问自己）：

| 检查项 | 你的函数 |
| --- | --- |
| 做了几件事？ | 爬取 + 解析 + 输出 = 3 件 ❌ |
| 改一个配置需要改几个地方？ | max\_pages/timeout/user\_agent 散落在代码里 ❌ |
| 能不能独立测试某个环节？ | 不能，都耦合在一起 ❌ |

#### 2\. 使用"单次提交只做一件事"作为边界

```bash
# 敏捷开发的工作流
git commit -m "feat: 实现基础爬取功能"      # 第一版，面条式没关系

# 然后立即开一个新分支重构
git checkout -b refactor/extract-classes
git commit -m "refactor: 提取 Config 类"
git commit -m "refactor: 提取 Fetcher 接口"
git commit -m "refactor: 提取 LinkExtractor"
```

**关键**：敏捷不等于不重构。快速交付后，预留 20% 时间做"技术还债"。

---

### 二、刻意练习 - 渐进式重构训练

#### 练习 1：职责识别训练

拿你现有的面条代码，用荧光笔/注释标出不同职责：

```python
def process_order(order_data):
    # ====== 职责A: 数据验证 ======
    if not order_data.get('user_id'):
        raise ValueError("user_id required")
    if order_data.get('amount', 0) <= 0:
        raise ValueError("invalid amount")

    # ====== 职责B: 数据转换 ======
    user = User.objects.get(id=order_data['user_id'])
    order = Order(
        user=user,
        amount=order_data['amount'],
        status='pending'
    )

    # ====== 职责C: 持久化 ======
    order.save()

    # ====== 职责D: 通知 ======
    send_email(user.email, "Order created")
    notify_slack(f"New order: {order.id}")
```

**训练**：把这 4 个职责分别提取成类。

#### 练习 2：依赖注入训练

从非 OOP 代码开始，一步步改造：

```python
# 阶段 0: 硬编码（面条式）
class ReportGenerator:
    def generate(self):
        data = requests.get("https://api.example.com/data").json()
        # ... 处理逻辑

# 阶段 1: 提取参数
class ReportGenerator:
    def __init__(self, api_url: str):
        self.api_url = api_url

    def generate(self):
        data = requests.get(self.api_url).json()

# 阶段 2: 依赖接口
class DataProvider(ABC):
    @abstractmethod
    def get_data(self) -> dict: pass

class ApiDataProvider(DataProvider):
    def __init__(self, url: str):
        self.url = url

    def get_data(self) -> dict:
        return requests.get(self.url).json()

class ReportGenerator:
    def __init__(self, provider: DataProvider):
        self.provider = provider

    def generate(self):
        data = self.provider.get_data()  # 不关心数据从哪来
```

**每天挑一个函数，从阶段 0 改造到阶段 2**，两周后你会有质的飞跃。

---

### 三、代码审查清单 - 培养 OOP 直觉

每次 PR 之前，用这个清单自查：

```markdown
## OOP 自检清单

### 类设计
- [ ] 类名是名词（不是动词）
- [ ] 每个类只有一个改变的理由（SRP）
- [ ] 类的方法数量 ≤ 7 个（人类短期记忆极限）
- [ ] 公开方法 ≤ 5 个

### 方法设计
- [ ] 方法只做一件事
- [ ] 参数 ≤ 4 个（超过就用对象封装）
- [ ] 不嵌套超过 3 层

### 依赖关系
- [ ] 依赖抽象（接口/ABC），不依赖具体实现
- [ ] 可以轻易 mock 依赖用于测试
- [ ] 不依赖全局状态

### 可测试性
- [ ] 可以不启动数据库/网络进行单元测试
- [ ] 测试不需要读取文件系统
```

---

### 四、推荐学习路径

#### 第一周：掌握基础模式

| 模式  | 适用场景 | 练习项目 |
| --- | --- | --- |
| 策略模式 | 多种算法可互换 | 实现支持多种支付方式的订单系统 |
| 工厂模式 | 创建对象逻辑复杂 | 实现支持多种数据库的连接器 |
| 适配器模式 | 接口不兼容 | 为第三方 SDK 统一接口 |

#### 第二周：重构 Kata

每天做 30 分钟重构练习：

```python
# 重构 Kata 示例：将这个面条代码改造成 OOP
def handle_booking(user_id, room_id, start, end):
    # 验证用户
    user = db.query(User).get(user_id)
    if not user or user.status != 'active':
        return {'error': 'invalid user'}

    # 验证房间
    room = db.query(Room).get(room_id)
    if not room or room.status != 'available':
        return {'error': 'invalid room'}

    # 检查时间冲突
    existing = db.query(Booking).filter(
        Booking.room_id == room_id,
        Booking.start < end,
        Booking.end > start
    ).first()
    if existing:
        return {'error': 'time conflict'}

    # 创建预订
    booking = Booking(user_id=user_id, room_id=room_id, start=start, end=end)
    db.add(booking)
    db.commit()

    # 发通知
    send_email(user.email, f'Booked {room.name}')
    return {'booking_id': booking.id}
```

**目标**：提取出 `UserValidator`、`RoomValidator`、`ConflictChecker`、`BookingCreator`、`NotificationService`

---

### 五、实战项目建议

做一个需要多个组件协作的项目，比如：

#### 项目：简易版 Git

```text
你的任务：实现一个简化版 Git

强制要求：
1. 必须有 Repository、Commit、Branch、Remote 等概念类
2. 支持 push/pull 时必须可以切换协议（HTTP/SSH）
3. 必须可以 mock 网络层进行测试
4. 配置、存储、网络传输必须分离

验收标准：
- 添加新的传输协议不需要修改核心逻辑
- 可以用假文件系统测试
```

---

### 六、敏捷环境下的实用技巧

#### 技巧 1：Triage 时间块

```text
周一、三、五：写面条代码，快速交付功能
周二、四：重构昨天写的面条代码
```

#### 技巧 2：Boy Scout Rule

> "Always leave the code better than you found it."

每次修改文件时，至少做一个小重构：

-   提取一个常量
    
-   拆分一个长函数
    
-   添加一个类型注解
    

#### 技巧 3：Code Review 聚焦 OOP

团队 Code Review 时，不只看"能不能跑"，还要问：

-   "这个类还能再拆吗？"
    
-   "如果要换掉这个依赖，需要改几个地方？"
    
-   "能写单元测试吗？"
    

---

### 七、推荐资源

| 资源  | 为什么值得读 |
| --- | --- |
| 《重构》- Martin Fowler | 教你"怎么改"，不是"怎么写" |
| 《敏捷软件工艺》 | 敏捷不等于乱写代码 |
| Refactoring.guru | 图解设计模式，有代码对比 |
| 你的 main\_oop.py | 已经是很好的 OOP 实践了！ |

---

## 总结

| 阶段  | 行动  |
| --- | --- |
| **写代码时** | 先跑起来，然后问"这个函数做了几件事？" |
| **Review 时** | 用上面的自检清单 |
| **重构时** | 每次只改一个类，立即测试 |
| **日常** | 每天做 30 分钟重构 Kata |

你现在的 `main_oop.py` 已经有很好的 OOP 意识了。继续保持这种"先抽象再实现"的思维，面条代码会越来越少。

> **抽象不是一开始就能想到的，而是当你发现代码开始变得"难以改变"时，通过重构逐步分离职责。**