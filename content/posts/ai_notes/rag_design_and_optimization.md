+++
date = '2026-08-28T14:00:00+08:00'
draft = false
title = 'RAG 系统设计与优化实践'
tags = ["AI", "RAG", "LangChain"]
categories = ["AI 技术笔记"]
description = "从基础 RAG 架构到工程优化实践，涵盖文档分块、检索策略、Prompt 设计等关键环节的踩坑与总结。"
+++

## 什么是 RAG

RAG（Retrieval-Augmented Generation，检索增强生成）是一种将外部知识库与大语言模型结合的技术架构。核心思路很简单：先检索，再生成。

LLM 本身有两个硬伤——知识截止日期的限制和幻觉问题。RAG 通过在生成前注入相关的外部知识，让 LLM 的回答"有据可查"，从而缓解这两个问题。

```
用户提问 → 检索相关文档片段 → 将片段 + 问题一起喂给 LLM → LLM 基于上下文生成回答
```

这个架构看似简单，但工程落地的过程中，每个环节都有大量的细节影响最终效果。本文记录我在实际项目中踩过的坑和优化经验。

## 文档处理：分块策略是第一道关

### 固定长度分块

最简单的分块方式是按固定字符数切分，比如每 500 个字符一刀，相邻块之间保留 50-100 字符的重叠（overlap）。实现简单，但问题也很明显：

- 语义可能在切分边界处被截断，一个完整的论述被切成两半
- 表格、列表等结构化内容被破坏
- 不同文档的最优 chunk size 差异很大

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,
    chunk_overlap=50,
    separators=["\n\n", "\n", "。", "，", " "]  # 按语义优先级切分
)
chunks = splitter.split_text(document)
```

`RecursiveCharacterTextSplitter` 比纯固定长度切分好很多——它会优先在段落、句子等自然边界处切分，而不是硬切到一半。

### 语义分块

更进阶的做法是根据文档结构进行语义分块：

- **按标题层级切分**：先用 Markdown 的标题（`#`, `##`, `###`）作为天然分割点，每个章节作为一个块
- **按段落聚合**：将相邻的短段落合并成一个块，避免碎片化
- **混合策略**：先按结构切分，对过长的块再递归细分

### chunk size 怎么选

这是一个需要实验的参数，没有万能公式。经验法则：

| chunk size | 适用场景 | 风险 |
|-----------|---------|------|
| 200-300 | 短问答、精确检索 | 上下文可能不完整 |
| 500-800 | 通用问答 | 平衡选择 |
| 1000+ | 需要长上下文的问题 | 检索精度下降，噪音增多 |

建议从 500 开始，根据实际效果调整。关键指标是：检索到的块是否包含了回答问题所需的信息。

## 检索策略：从单路召回到多路召回

### 基础向量检索

最常见的做法是用 Embedding 模型把文档块转成向量，存入向量数据库，查询时计算 query 与文档向量的余弦相似度，取 Top-K。

```python
from langchain.embeddings import OpenAIEmbeddings
from langchain.vectorstores import FAISS

embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_texts(chunks, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})
docs = retriever.get_relevant_documents(query)
```

这种方式对语义相似的问题效果不错，但对关键词精确匹配较弱。比如用户问"ABC-2024-001 号订单的状态"，向量检索可能找到"订单查询"相关的通用文档，而不是精确匹配到这个订单号。

### 混合检索：向量 + 关键词

将向量检索和 BM25 关键词检索结合，用 RRF（Reciprocal Rank Fusion）或其他策略融合结果：

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

bm25_retriever = BM25Retriever.from_texts(chunks, k=5)
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

ensemble = EnsembleRetriever(
    retrievers=[bm25_retriever, vector_retriever],
    weights=[0.4, 0.6]  # 向量检索权重更高
)
```

混合检索在以下场景特别有效：
- 用户问题包含专有名词、编号等精确关键词
- 文档中有大量结构化数据（表格、代码等）

### 查询改写

用户的原始提问不一定适合直接拿去做检索。通过 LLM 对 query 进行改写，可以显著提升召回率：

- **展开缩写**：用户说"RAG"，改写为"Retrieval Augmented Generation RAG 检索增强生成"
- **多角度改写**：把一个问题改写为 2-3 个不同表述，分别检索后合并结果
- **假设性文档**：让 LLM 先生成一个"假设性答案"，用这个答案去检索（HyDE 策略）

## Prompt 设计：让 LLM 老老实实基于上下文回答

检索到了相关文档，下一步是设计好 Prompt，让 LLM 基于这些文档回答问题，而不是自由发挥。

```python
template = """基于以下参考文档回答用户的问题。
如果参考文档中没有相关信息，请明确说明"根据现有资料无法回答此问题"，不要编造答案。

参考文档：
{context}

用户问题：{question}

回答："""
```

几个关键原则：

1. **明确约束**：告诉 LLM 只基于提供的上下文回答，不知道就说不知道
2. **要求引用**：让 LLM 标注答案来源（"根据文档 X"），便于用户验证
3. **控制长度**：避免 LLM 过度展开，要求简洁、结构化地回答
4. **处理冲突**：当多个文档片段的说法不一致时，要求 LLM 指出分歧

## 效果评估：怎么知道 RAG 系统好不好

RAG 系统的评估需要分两个维度：

### 检索质量
- **Recall@K**：Top-K 检索结果中包含了正确答案相关文档的比例
- **MRR（Mean Reciprocal Rank）**：第一个相关结果排在第几位

### 生成质量
- **忠实度（Faithfulness）**：生成的答案是否忠实于检索到的上下文，有没有编造
- **相关性（Relevance）**：答案是否回答了用户的问题
- **完整度（Completeness）**：答案是否覆盖了问题的所有方面

可以用 RAGAS 等框架进行自动化评估，也可以人工抽样检查。实际项目中，建议两者结合——自动化评估做日常监控，人工评估做深度分析。

## 总结：RAG 优化的核心 checklist

| 环节 | 关键决策 | 常见坑 |
|------|---------|--------|
| 文档解析 | 选择合适的 Loader，保留结构信息 | PDF 解析丢失表格和格式 |
| 分块策略 | chunk_size、overlap、分割方式 | 语义在切分边界处断裂 |
| Embedding | 选择适合领域的 Embedding 模型 | 中文场景用英文模型效果差 |
| 检索策略 | 单路 vs 混合检索，Top-K 大小 | 只看向量检索，忽略关键词匹配 |
| Prompt | 约束 LLM 行为，要求引用来源 | LLM 自由发挥产生幻觉 |
| 评估 | 检索和生成分别评估 | 只看最终回答，不排查是哪个环节出了问题 |

RAG 不是一个"搭完就不管"的系统，而是一个需要持续迭代的工程。每个环节的微调都可能带来显著的效果变化，关键在于建立一套可量化的评估流程，用数据驱动优化决策。
