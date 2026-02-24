# 进阶模块三：RAG 高级实践

> 构建生产级的检索增强生成系统

---

## 第32章：RAG 高级技术

### 32.1 查询处理优化

```
用户的原始查询往往不是最佳的检索查询

查询改写（Query Rewriting）：

┌─────────────────────────────────────────────────────────┐
│                  查询改写策略                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 查询扩展（Query Expansion）                         │
│  ─────────────────────────────                          │
│  原始查询："如何优化 Python 代码？"                    │
│  扩展后：                                               │
│  • "Python 性能优化技巧"                               │
│  • "Python 代码加速方法"                               │
│  • "提高 Python 运行效率"                              │
│                                                         │
│  实现：用 LLM 生成同义查询                             │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  2. HyDE（Hypothetical Document Embedding）             │
│  ───────────────────────────────────────────            │
│  思想：先让 LLM 生成假设性答案，再用答案检索           │
│                                                         │
│  流程：                                                 │
│  查询 → LLM 生成假设答案 → 嵌入假设答案 → 检索        │
│                                                         │
│  优势：假设答案与真实文档更相似                        │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  3. Step-back Prompting                                 │
│  ───────────────────────                                │
│  思想：先问一个更一般的问题                            │
│                                                         │
│  原始："Python 3.11 的新特性？"                        │
│  Step-back："Python 版本更新历史和重要特性"           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```python
# HyDE 实现示例
from openai import OpenAI

client = OpenAI()

def hyde_query(query: str) -> str:
    """使用 HyDE 策略改写查询"""
    
    # 1. 让 LLM 生成假设性文档
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"""请写一段简短的文档来回答以下问题。
不需要完全准确，只需要生成一个合理的假设性回答。

问题：{query}

假设性文档："""
        }],
        temperature=0.7
    )
    
    hypothetical_doc = response.choices[0].message.content
    
    # 2. 使用假设性文档进行向量检索
    # embedding = get_embedding(hypothetical_doc)
    # results = vector_store.search(embedding)
    
    return hypothetical_doc


# 多查询改写
def multi_query_rewrite(query: str, n: int = 3) -> list[str]:
    """生成多个查询变体"""
    
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{
            "role": "user",
            "content": f"""基于以下查询，生成 {n} 个语义相似但表达不同的查询变体。
每行一个查询。

原始查询：{query}

变体查询："""
        }]
    )
    
    queries = response.choices[0].message.content.strip().split("\n")
    return [q.strip() for q in queries if q.strip()]
```

### 32.2 混合检索（Hybrid Search）

```
结合向量检索和关键词检索的优势

┌─────────────────────────────────────────────────────────┐
│                    混合检索架构                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│                    用户查询                             │
│                       │                                 │
│           ┌───────────┴───────────┐                    │
│           ▼                       ▼                    │
│     ┌──────────┐           ┌──────────┐               │
│     │ 向量检索 │           │关键词检索│               │
│     │(语义相似)│           │(精确匹配)│               │
│     └────┬─────┘           └────┬─────┘               │
│          │                      │                      │
│          │    ┌────────────┐    │                      │
│          └───▶│  分数融合  │◀───┘                      │
│               └─────┬──────┘                           │
│                     │                                  │
│                     ▼                                  │
│              ┌──────────────┐                          │
│              │  最终结果排序 │                          │
│              └──────────────┘                          │
│                                                         │
│  融合方法：                                             │
│  • RRF（Reciprocal Rank Fusion）                       │
│  • 加权平均                                            │
│  • 学习排序                                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```python
from typing import List, Tuple
import numpy as np

def reciprocal_rank_fusion(
    results_list: List[List[Tuple[str, float]]],
    k: int = 60
) -> List[Tuple[str, float]]:
    """
    RRF 融合多个检索结果
    
    results_list: 多个检索结果列表，每个是 [(doc_id, score), ...]
    k: RRF 常数，通常取 60
    """
    doc_scores = {}
    
    for results in results_list:
        for rank, (doc_id, _) in enumerate(results):
            if doc_id not in doc_scores:
                doc_scores[doc_id] = 0
            # RRF 公式：1 / (k + rank)
            doc_scores[doc_id] += 1 / (k + rank + 1)
    
    # 按分数排序
    sorted_docs = sorted(doc_scores.items(), key=lambda x: x[1], reverse=True)
    return sorted_docs


def hybrid_search(
    query: str,
    vector_results: List[Tuple[str, float]],
    keyword_results: List[Tuple[str, float]],
    alpha: float = 0.5
) -> List[Tuple[str, float]]:
    """
    加权混合检索
    
    alpha: 向量检索权重 (0-1)
    """
    # 归一化分数
    def normalize(results):
        if not results:
            return {}
        scores = [s for _, s in results]
        min_s, max_s = min(scores), max(scores)
        if max_s == min_s:
            return {doc: 1.0 for doc, _ in results}
        return {doc: (s - min_s) / (max_s - min_s) for doc, s in results}
    
    vector_norm = normalize(vector_results)
    keyword_norm = normalize(keyword_results)
    
    # 合并文档
    all_docs = set(vector_norm.keys()) | set(keyword_norm.keys())
    
    # 计算混合分数
    hybrid_scores = {}
    for doc in all_docs:
        v_score = vector_norm.get(doc, 0)
        k_score = keyword_norm.get(doc, 0)
        hybrid_scores[doc] = alpha * v_score + (1 - alpha) * k_score
    
    return sorted(hybrid_scores.items(), key=lambda x: x[1], reverse=True)
```

### 32.3 重排序（Re-ranking）

```
检索后用更精确的模型重新排序

┌─────────────────────────────────────────────────────────┐
│                    重排序流程                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  第一阶段：粗排（检索）                                 │
│  • 快速检索 Top 50-100 候选文档                        │
│  • 使用向量检索或 BM25                                 │
│  • 追求召回率                                          │
│                                                         │
│  第二阶段：精排（重排序）                               │
│  • 用更精确的模型对候选重新打分                        │
│  • Cross-Encoder 逐对打分                              │
│  • 选出 Top 5-10 最终结果                              │
│  • 追求准确率                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘

重排序模型：

1. Cross-Encoder
   • 将查询和文档一起输入模型
   • 输出相关性分数
   • 准确但较慢

2. 常用模型
   • Cohere Rerank
   • BGE Reranker
   • ms-marco-MiniLM-L-12-v2
```

```python
# 使用 Cohere Rerank
import cohere

co = cohere.Client('your-api-key')

def rerank_with_cohere(
    query: str,
    documents: List[str],
    top_n: int = 5
) -> List[dict]:
    """使用 Cohere Rerank 重排序"""
    
    results = co.rerank(
        model="rerank-english-v3.0",
        query=query,
        documents=documents,
        top_n=top_n
    )
    
    return [
        {
            "index": r.index,
            "score": r.relevance_score,
            "document": documents[r.index]
        }
        for r in results.results
    ]


# 使用开源 Cross-Encoder
from sentence_transformers import CrossEncoder

def rerank_with_cross_encoder(
    query: str,
    documents: List[str],
    model_name: str = "cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n: int = 5
) -> List[Tuple[str, float]]:
    """使用 Cross-Encoder 重排序"""
    
    model = CrossEncoder(model_name)
    
    # 构造 query-document 对
    pairs = [[query, doc] for doc in documents]
    
    # 批量打分
    scores = model.predict(pairs)
    
    # 排序
    doc_scores = list(zip(documents, scores))
    doc_scores.sort(key=lambda x: x[1], reverse=True)
    
    return doc_scores[:top_n]
```

### 32.4 上下文压缩

```
检索到的内容可能包含很多无关信息

上下文压缩：只保留与查询相关的部分

┌─────────────────────────────────────────────────────────┐
│                    压缩前 vs 压缩后                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  压缩前（原始检索结果）：                               │
│  ┌────────────────────────────────────────────────┐    │
│  │ Python 是一种高级编程语言，由 Guido van Rossum │    │
│  │ 于 1991 年发布。Python 的设计哲学强调代码的可 │    │
│  │ 读性。【Python 的装饰器是一种设计模式，用于   │    │
│  │ 修改函数或类的行为。】装饰器使用 @ 符号...    │    │
│  │ Python 还支持多种编程范式...                  │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  查询："什么是 Python 装饰器？"                        │
│                                                         │
│  压缩后：                                               │
│  ┌────────────────────────────────────────────────┐    │
│  │ Python 的装饰器是一种设计模式，用于修改函数或 │    │
│  │ 类的行为。装饰器使用 @ 符号...                │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
│  优势：                                                 │
│  • 减少 Token 消耗                                     │
│  • 减少噪音，提高回答质量                              │
│  • 可以在上下文中放入更多相关内容                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```python
from openai import OpenAI

client = OpenAI()

def compress_context(
    query: str,
    documents: List[str],
    max_tokens_per_doc: int = 200
) -> List[str]:
    """压缩文档，只保留与查询相关的部分"""
    
    compressed = []
    
    for doc in documents:
        response = client.chat.completions.create(
            model="gpt-4o-mini",
            messages=[{
                "role": "user",
                "content": f"""从以下文档中提取与查询相关的信息。
只保留直接相关的内容，删除无关部分。
如果文档与查询完全无关，返回"无相关内容"。

查询：{query}

文档：
{doc}

相关内容："""
            }],
            max_tokens=max_tokens_per_doc
        )
        
        extracted = response.choices[0].message.content.strip()
        if extracted != "无相关内容":
            compressed.append(extracted)
    
    return compressed


# LongContextReorder: 重新排列长上下文
def reorder_for_long_context(documents: List[str]) -> List[str]:
    """
    重新排列文档顺序，将最相关的放在开头和结尾
    （LLM 对开头和结尾的信息记忆更好）
    """
    if len(documents) <= 2:
        return documents
    
    # 假设文档已按相关性排序
    reordered = []
    
    # 交替放置：最相关的在开头和结尾
    for i, doc in enumerate(documents):
        if i % 2 == 0:
            reordered.insert(0, doc)  # 偶数位放开头
        else:
            reordered.append(doc)      # 奇数位放结尾
    
    return reordered
```

---

## 第33章：RAG 系统架构

### 33.1 生产级 RAG 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                     生产级 RAG 系统架构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                      用户请求                            │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    API Gateway                           │   │
│  │            (认证、限流、负载均衡)                        │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   查询处理服务                           │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │   │
│  │  │查询改写 │  │意图识别 │  │安全过滤 │                 │   │
│  │  └─────────┘  └─────────┘  └─────────┘                 │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    检索服务                              │   │
│  │  ┌───────────────┐  ┌───────────────┐                   │   │
│  │  │   向量检索    │  │  关键词检索   │                   │   │
│  │  └───────┬───────┘  └───────┬───────┘                   │   │
│  │          └─────────┬────────┘                           │   │
│  │                    ▼                                     │   │
│  │            ┌─────────────┐                              │   │
│  │            │  结果融合   │                              │   │
│  │            └──────┬──────┘                              │   │
│  │                   ▼                                      │   │
│  │            ┌─────────────┐                              │   │
│  │            │   重排序    │                              │   │
│  │            └─────────────┘                              │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    生成服务                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │ Prompt 构造 │  │  LLM 调用   │  │ 流式输出    │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   后处理服务                             │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│  │  │ 引用生成    │  │ 事实核查    │  │ 安全过滤    │     │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                    支撑服务                              │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │   │
│  │  │向量数据库│  │缓存服务 │  │日志监控 │  │反馈收集 │   │   │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 33.2 数据处理管道

```
文档处理是 RAG 系统的基础

┌─────────────────────────────────────────────────────────┐
│                    数据处理管道                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 数据采集                                            │
│  ───────────                                            │
│  • 定时爬取/同步                                       │
│  • 增量更新                                            │
│  • 变更检测                                            │
│                                                         │
│  2. 文档解析                                            │
│  ───────────                                            │
│  • PDF → 文本（PyPDF、pdfplumber）                    │
│  • Word → 文本（python-docx）                         │
│  • HTML → 文本（BeautifulSoup）                       │
│  • 表格提取                                            │
│  • OCR（如需要）                                       │
│                                                         │
│  3. 文档清洗                                            │
│  ───────────                                            │
│  • 去除噪音（页眉页脚、广告）                          │
│  • 格式标准化                                          │
│  • 编码处理                                            │
│                                                         │
│  4. 元数据提取                                          │
│  ──────────────                                         │
│  • 标题、作者、日期                                    │
│  • 文档类型、来源                                      │
│  • 自动标签/分类                                       │
│                                                         │
│  5. 分块处理                                            │
│  ───────────                                            │
│  • 选择合适的分块策略                                  │
│  • 保持语义完整性                                      │
│  • 添加上下文重叠                                      │
│                                                         │
│  6. 向量化                                              │
│  ─────────                                              │
│  • 批量处理                                            │
│  • 并行加速                                            │
│  • 错误重试                                            │
│                                                         │
│  7. 索引存储                                            │
│  ───────────                                            │
│  • 向量索引                                            │
│  • 全文索引                                            │
│  • 元数据索引                                          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```python
# 生产级文档处理管道示例
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime
import hashlib

@dataclass
class Document:
    content: str
    metadata: dict
    doc_id: str
    chunks: List["Chunk"] = None
    
    @staticmethod
    def generate_id(content: str, source: str) -> str:
        """生成文档唯一 ID"""
        return hashlib.md5(f"{source}:{content[:100]}".encode()).hexdigest()

@dataclass
class Chunk:
    content: str
    chunk_id: str
    doc_id: str
    metadata: dict
    embedding: Optional[List[float]] = None

class DocumentPipeline:
    def __init__(self, embedding_model, vector_store, text_store):
        self.embedding_model = embedding_model
        self.vector_store = vector_store
        self.text_store = text_store
    
    def process_document(self, raw_content: str, source: str, metadata: dict) -> Document:
        """处理单个文档"""
        
        # 1. 清洗
        cleaned = self._clean_content(raw_content)
        
        # 2. 创建文档
        doc_id = Document.generate_id(cleaned, source)
        doc = Document(
            content=cleaned,
            metadata={**metadata, "source": source, "processed_at": datetime.now().isoformat()},
            doc_id=doc_id
        )
        
        # 3. 分块
        doc.chunks = self._chunk_document(doc)
        
        # 4. 向量化
        self._embed_chunks(doc.chunks)
        
        # 5. 存储
        self._store_document(doc)
        
        return doc
    
    def _clean_content(self, content: str) -> str:
        """清洗文档内容"""
        # 实现清洗逻辑
        return content.strip()
    
    def _chunk_document(self, doc: Document) -> List[Chunk]:
        """分块"""
        # 实现分块逻辑
        chunks = []
        # ...
        return chunks
    
    def _embed_chunks(self, chunks: List[Chunk]):
        """批量向量化"""
        contents = [c.content for c in chunks]
        embeddings = self.embedding_model.embed_batch(contents)
        for chunk, emb in zip(chunks, embeddings):
            chunk.embedding = emb
    
    def _store_document(self, doc: Document):
        """存储文档和向量"""
        # 存储原文
        self.text_store.save(doc.doc_id, doc.content, doc.metadata)
        
        # 存储向量
        for chunk in doc.chunks:
            self.vector_store.upsert(
                id=chunk.chunk_id,
                vector=chunk.embedding,
                metadata={**chunk.metadata, "doc_id": doc.doc_id}
            )
```

### 33.3 缓存策略

```
缓存是提升 RAG 性能的关键

┌─────────────────────────────────────────────────────────┐
│                    缓存层次                             │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  L1: 查询结果缓存                                       │
│  ─────────────────                                      │
│  • 缓存完整的问答对                                    │
│  • 适合高频重复问题                                    │
│  • TTL: 较短（如 1 小时）                              │
│                                                         │
│  L2: 检索结果缓存                                       │
│  ─────────────────                                      │
│  • 缓存检索到的文档                                    │
│  • 查询向量 → 文档列表                                │
│  • TTL: 中等（如 24 小时）                             │
│                                                         │
│  L3: 嵌入向量缓存                                       │
│  ─────────────────                                      │
│  • 缓存文本的嵌入向量                                  │
│  • 避免重复调用 Embedding API                          │
│  • TTL: 较长（直到文档更新）                           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```python
import redis
import json
import hashlib
from typing import Optional, List

class RAGCache:
    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
    
    def _hash_query(self, query: str) -> str:
        """生成查询的哈希键"""
        return hashlib.md5(query.lower().strip().encode()).hexdigest()
    
    # === 查询结果缓存 ===
    def get_answer(self, query: str) -> Optional[str]:
        """获取缓存的答案"""
        key = f"answer:{self._hash_query(query)}"
        result = self.redis.get(key)
        return result.decode() if result else None
    
    def set_answer(self, query: str, answer: str, ttl: int = 3600):
        """缓存答案"""
        key = f"answer:{self._hash_query(query)}"
        self.redis.setex(key, ttl, answer)
    
    # === 检索结果缓存 ===
    def get_retrieval(self, query: str) -> Optional[List[dict]]:
        """获取缓存的检索结果"""
        key = f"retrieval:{self._hash_query(query)}"
        result = self.redis.get(key)
        return json.loads(result) if result else None
    
    def set_retrieval(self, query: str, docs: List[dict], ttl: int = 86400):
        """缓存检索结果"""
        key = f"retrieval:{self._hash_query(query)}"
        self.redis.setex(key, ttl, json.dumps(docs))
    
    # === 语义缓存（近似查询匹配）===
    def get_similar_answer(self, query_embedding: List[float], threshold: float = 0.95):
        """
        查找语义相似的缓存答案
        需要配合向量数据库使用
        """
        # 实现：在向量数据库中搜索相似的历史查询
        pass


# 使用示例
class CachedRAG:
    def __init__(self, rag_pipeline, cache: RAGCache):
        self.rag = rag_pipeline
        self.cache = cache
    
    def query(self, question: str) -> str:
        # 1. 检查答案缓存
        cached_answer = self.cache.get_answer(question)
        if cached_answer:
            return cached_answer
        
        # 2. 检查检索缓存
        cached_docs = self.cache.get_retrieval(question)
        if cached_docs:
            # 直接用缓存的文档生成答案
            answer = self.rag.generate(question, cached_docs)
        else:
            # 完整检索流程
            docs = self.rag.retrieve(question)
            self.cache.set_retrieval(question, docs)
            answer = self.rag.generate(question, docs)
        
        # 3. 缓存答案
        self.cache.set_answer(question, answer)
        
        return answer
```

---

## 第34章：RAG 评估与优化

### 34.1 评估框架

```
RAG 评估的三个维度：

┌─────────────────────────────────────────────────────────┐
│                    RAG 评估框架                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 检索质量                                            │
│  ───────────                                            │
│  • 上下文精确度（Context Precision）                   │
│    检索到的内容中有多少是相关的                        │
│                                                         │
│  • 上下文召回率（Context Recall）                      │
│    相关内容有多少被检索到                              │
│                                                         │
│  • 上下文相关性（Context Relevancy）                   │
│    检索内容与问题的相关程度                            │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  2. 生成质量                                            │
│  ───────────                                            │
│  • 答案相关性（Answer Relevancy）                      │
│    答案是否回答了用户的问题                            │
│                                                         │
│  • 忠实度（Faithfulness）                              │
│    答案是否基于检索内容，不产生幻觉                    │
│                                                         │
│  • 答案正确性（Answer Correctness）                    │
│    与标准答案对比的准确度                              │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  3. 端到端质量                                          │
│  ──────────────                                         │
│  • 用户满意度                                          │
│  • 任务完成率                                          │
│  • 响应时间                                            │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

```python
# 使用 RAGAS 评估框架
# pip install ragas

from ragas import evaluate
from ragas.metrics import (
    faithfulness,
    answer_relevancy,
    context_precision,
    context_recall,
)
from datasets import Dataset

# 准备评估数据
eval_data = {
    "question": [
        "什么是机器学习？",
        "Python 有哪些数据类型？"
    ],
    "answer": [
        "机器学习是人工智能的一个分支...",
        "Python 的基本数据类型包括..."
    ],
    "contexts": [
        ["机器学习是一种通过数据学习规律的技术..."],
        ["Python 支持多种数据类型，包括 int、float..."]
    ],
    "ground_truth": [
        "机器学习是人工智能的一个分支，通过算法从数据中学习规律",
        "Python 的数据类型包括整数、浮点数、字符串、列表、字典等"
    ]
}

dataset = Dataset.from_dict(eval_data)

# 运行评估
results = evaluate(
    dataset,
    metrics=[
        faithfulness,
        answer_relevancy,
        context_precision,
        context_recall,
    ]
)

print(results)
# {'faithfulness': 0.92, 'answer_relevancy': 0.88, ...}
```

### 34.2 常见问题与解决方案

```
┌─────────────────────────────────────────────────────────┐
│                 RAG 常见问题与解决方案                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  问题 1：检索不到相关内容                               │
│  ───────────────────────────                            │
│  症状：答案说"没有相关信息"但文档中有                  │
│                                                         │
│  可能原因：                                             │
│  • 分块太大/太小                                       │
│  • Embedding 模型不适合                                │
│  • 查询与文档表述差异大                                │
│                                                         │
│  解决方案：                                             │
│  • 调整分块策略（尝试 300-500 tokens）                 │
│  • 换更好的 Embedding 模型                             │
│  • 添加查询改写                                        │
│  • 使用混合检索                                        │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  问题 2：检索到但答案不对                               │
│  ────────────────────────                               │
│  症状：检索内容正确，但 LLM 回答错误                   │
│                                                         │
│  可能原因：                                             │
│  • 检索内容太多，关键信息被淹没                        │
│  • Prompt 不清晰                                       │
│  • LLM 产生幻觉                                        │
│                                                         │
│  解决方案：                                             │
│  • 减少检索数量，提高精度                              │
│  • 添加重排序                                          │
│  • 优化 Prompt，强调"只根据资料回答"                  │
│  • 使用上下文压缩                                      │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  问题 3：答案包含幻觉                                   │
│  ─────────────────────                                  │
│  症状：答案中有检索内容中不存在的信息                  │
│                                                         │
│  可能原因：                                             │
│  • LLM 补充了训练知识                                  │
│  • 检索内容不足以回答问题                              │
│                                                         │
│  解决方案：                                             │
│  • 在 Prompt 中强调"只用提供的资料"                   │
│  • 要求引用来源                                        │
│  • 降低 temperature                                    │
│  • 添加事实核查步骤                                    │
│  • 允许说"不知道"                                     │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  问题 4：响应太慢                                       │
│  ────────────────                                       │
│  可能原因：                                             │
│  • Embedding 调用慢                                    │
│  • 检索范围太大                                        │
│  • LLM 生成太长                                        │
│                                                         │
│  解决方案：                                             │
│  • 添加缓存                                            │
│  • 使用更快的向量索引                                  │
│  • 流式输出                                            │
│  • 异步处理                                            │
│  • 限制最大生成长度                                    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 34.3 A/B 测试与迭代

```python
from dataclasses import dataclass
from typing import Dict, List
import random

@dataclass
class RAGExperiment:
    name: str
    config: Dict
    metrics: Dict = None

class RAGABTest:
    def __init__(self):
        self.experiments: Dict[str, RAGExperiment] = {}
        self.results: Dict[str, List[Dict]] = {}
    
    def add_experiment(self, name: str, config: Dict):
        """添加实验配置"""
        self.experiments[name] = RAGExperiment(name=name, config=config)
        self.results[name] = []
    
    def assign_experiment(self, user_id: str) -> str:
        """为用户分配实验组"""
        # 简单的随机分配
        return random.choice(list(self.experiments.keys()))
    
    def record_result(self, experiment_name: str, query: str, answer: str, 
                     user_feedback: int, latency: float):
        """记录实验结果"""
        self.results[experiment_name].append({
            "query": query,
            "answer": answer,
            "feedback": user_feedback,  # 1-5 分
            "latency": latency
        })
    
    def analyze(self) -> Dict:
        """分析实验结果"""
        analysis = {}
        for name, results in self.results.items():
            if not results:
                continue
            analysis[name] = {
                "sample_size": len(results),
                "avg_feedback": sum(r["feedback"] for r in results) / len(results),
                "avg_latency": sum(r["latency"] for r in results) / len(results),
            }
        return analysis


# 使用示例
ab_test = RAGABTest()

# 定义实验组
ab_test.add_experiment("baseline", {
    "chunk_size": 500,
    "top_k": 5,
    "rerank": False
})

ab_test.add_experiment("with_rerank", {
    "chunk_size": 500,
    "top_k": 10,
    "rerank": True
})

# 在请求处理中
def handle_query(user_id: str, query: str):
    # 分配实验组
    experiment = ab_test.assign_experiment(user_id)
    config = ab_test.experiments[experiment].config
    
    # 根据配置处理请求
    # ...
    
    # 记录结果
    ab_test.record_result(experiment, query, answer, feedback, latency)
```

---

## 本章小结

```
核心要点回顾：

1. 查询处理优化
   ├── 查询改写：扩展、HyDE、Step-back
   ├── 多查询：生成查询变体
   └── 目的：提高检索召回率

2. 高级检索技术
   ├── 混合检索：向量 + 关键词
   ├── 重排序：粗排 → 精排
   ├── 上下文压缩：只保留相关内容
   └── 目的：提高检索精度

3. 生产级架构
   ├── 分层架构：查询、检索、生成、后处理
   ├── 数据管道：采集、解析、分块、向量化
   ├── 缓存策略：多层缓存
   └── 目的：性能、可扩展性、可维护性

4. 评估与优化
   ├── 评估维度：检索质量、生成质量、端到端
   ├── 常见问题：检索失败、答案错误、幻觉、慢
   ├── A/B 测试：科学迭代
   └── 目的：持续改进
```

---

**下一章 → [第35章：模型微调基础](./11-fine-tuning.md)**
