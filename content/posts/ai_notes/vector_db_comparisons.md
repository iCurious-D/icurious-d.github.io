+++
date = '2026-08-30T10:00:00+08:00'
draft = false
title = '向量数据库选型与实践'
tags = ["AI", "向量数据库", "RAG"]
categories = ["AI 技术笔记"]
description = "主流向量数据库的选型对比与实践经验，涵盖 FAISS、Chroma、Milvus 等方案的特点、适用场景与性能考量。"
pinned = true
+++

## 为什么需要向量数据库

在 RAG、语义搜索等场景中，我们需要将文本转化为向量（Embedding），然后通过向量相似度检索找到语义相关的内容。传统的数据库（MySQL、Redis 等）擅长精确匹配，但做不了"这段话和那段话意思相近"这种语义级别的检索。
向量数据库就是为解决这个问题而生的——它专门为高维向量的存储和相似度检索而设计。

## 核心检索原理

向量数据库的核心操作是**近似最近邻搜索（ANN，Approximate Nearest Neighbor）**：

```
1. 文本 → Embedding 模型 → 高维向量（如 1536 维）
2. 向量存入向量数据库，附带原始文本和元数据
3. 查询时：query → Embedding → 在数据库中找到最相似的 Top-K 个向量
4. 返回对应的原始文本
```

相似度计算通常使用余弦相似度（Cosine Similarity）或欧氏距离（L2 Distance）。

## 主流向量数据库对比

### FAISS（Facebook AI Similarity Search）

**定位**：高性能向量检索库，Meta 开源。

```python
import faiss
import numpy as np

# 创建索引
dimension = 1536  # Embedding 维度
index = faiss.IndexFlatL2(dimension)  # 精确 L2 距离

# 添加向量
vectors = np.random.random((10000, dimension)).astype('float32')
index.add(vectors)

# 检索
query = np.random.random((1, dimension)).astype('float32')
distances, indices = index.search(query, k=5)
```

| 优点 | 缺点 |
|------|------|
| 速度极快，支持 GPU 加速 | 不是数据库，没有持久化和分布式能力 |
| 支持多种索引类型（Flat、IVF、HNSW 等） | 需要自己管理数据持久化 |
| 内存中运行，延迟极低 | 不适合大规模生产环境（单机限制） |
| 轻量，pip install 即可 | 没有元数据过滤功能 |

**适用场景**：本地开发、原型验证、数据量在百万级以下的中小规模场景。

### Chroma

**定位**：轻量级嵌入式向量数据库，专为 LLM 应用设计。

```python
import chromadb

client = chromadb.Client()
collection = client.create_collection("documents")

# 添加文档
collection.add(
    documents=["RAG 是一种检索增强生成技术", "向量数据库用于语义搜索"],
    metadatas=[{"source": "blog"}, {"source": "docs"}],
    ids=["doc1", "doc2"]
)

# 检索
results = collection.query(
    query_texts=["什么是 RAG"],
    n_results=2,
    where={"source": "blog"}  # 元数据过滤
)
```

| 优点 | 缺点 |
|------|------|
| 使用简单，API 友好 | 性能和可扩展性不如专业方案 |
| 支持元数据过滤 | 大规模数据下性能有瓶颈 |
| 嵌入式，无需单独部署 | 分布式支持有限 |
| 与 LangChain 集成良好 | 社区生态相对较新 |

**适用场景**：中小规模项目、快速原型开发、对部署复杂度敏感的场景。

### Milvus

**定位**：企业级分布式向量数据库。

| 优点 | 缺点 |
|------|------|
| 支持分布式部署，水平扩展 | 部署和运维复杂度高 |
| 支持十亿级向量的高效检索 | 资源消耗大 |
| 丰富的索引类型和调优选项 | 学习曲线较陡 |
| 支持标量过滤和混合查询 | 小规模场景"杀鸡用牛刀" |
| 活跃的社区和企业级支持 | 需要独立的运维资源 |

**适用场景**：大规模生产环境、数据量在亿级以上、对可靠性和扩展性有严格要求的企业项目。

### 其他值得关注的方案

| 方案 | 特点 |
|------|------|
| **Pinecone** | 全托管云服务，零运维，按量付费，适合不想自建基础设施的团队 |
| **Qdrant** | Rust 编写，性能优秀，支持丰富的过滤条件，API 设计现代 |
| **Weaviate** | 支持多模态（文本+图像），内置向量化模块，GraphQL 接口 |
| **pgvector** | PostgreSQL 扩展，如果你的系统已经用了 PG，可以零迁移成本加上向量能力 |

## 选型决策树

```
数据规模多大？
├── < 100 万条
│   ├── 需要快速原型 → Chroma
│   └── 追求极致性能 → FAISS
├── 100 万 - 1 亿条
│   ├── 不想运维 → Pinecone / Qdrant Cloud
│   └── 自部署 → Qdrant / Milvus Lite
└── > 1 亿条
    └── Milvus（分布式部署）
```

## 实践经验

### 经验 1：Embedding 维度要和数据库匹配

不同的 Embedding 模型输出不同维度的向量：

| 模型 | 维度 | 来源 |
|------|------|------|
| text-embedding-3-small | 1536 | OpenAI |
| text-embedding-3-large | 3072 | OpenAI |
| bge-large-zh | 1024 | BAAI |
| m3e-base | 768 | Moka AI |

创建向量索引时，维度参数必须与 Embedding 模型的输出维度一致，否则报错。

### 经验 2：索引类型影响性能

以 FAISS 为例，不同索引类型的取舍：

| 索引 | 精度 | 速度 | 内存 |
|------|------|------|------|
| IndexFlatL2 | 精确 | 慢（暴力搜索） | 高 |
| IndexIVFFlat | 近似 | 快 | 中 |
| IndexHNSW | 近似 | 很快 | 高 |

数据量小时用 Flat（精确），数据量大时切换到 IVF 或 HNSW（近似但快）。

### 经验 3：元数据过滤很重要

纯向量检索有时不够精确，结合元数据过滤可以显著提升效果：

```python
# 只在"产品手册"类别中检索
results = collection.query(
    query_texts=["如何退换货"],
    n_results=5,
    where={"category": "产品手册", "language": "zh"}
)
```

先通过元数据缩小范围，再做向量检索，既提升了精度，也降低了计算量。

### 经验 4：批量写入比逐条写入快得多

向向量数据库添加数据时，批量操作的性能远优于逐条插入：

```python
# 慢：逐条插入
for doc in documents:
    collection.add(documents=[doc], ids=[doc.id])

# 快：批量插入
collection.add(
    documents=[doc.text for doc in documents],
    ids=[doc.id for doc in documents],
    metadatas=[doc.metadata for doc in documents]
)
```

## 总结

向量数据库没有"最好"的，只有"最合适"的。选型时重点关注三个维度：**数据规模**决定了你需要什么级别的方案，**部署复杂度**决定了你的运维成本，**功能需求**（如元数据过滤、多模态支持）决定了具体的产品选择。

对于 AI 工程师的日常开发，建议从 Chroma 或 FAISS 起步，快速验证方案可行性，在数据量或性能需求增长时再考虑迁移到 Milvus 或云服务方案。
