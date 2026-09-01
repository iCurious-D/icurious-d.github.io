+++
date = '2026-08-29T14:00:00+08:00'
draft = false
title = 'LangChain 核心概念速查手册'
tags = ["AI", "LangChain", "LangGraph"]
categories = ["AI 技术笔记"]
description = "LangChain 生态核心概念速查，覆盖 Chain、Retriever、Agent、Memory 等关键组件的使用场景与最佳实践。"
+++

## LangChain 是什么

LangChain 是一个用于构建 LLM 应用的开发框架，核心思想是**把 LLM 的能力通过可组合的模块串联成完整的应用链路**。它不是一个模型，而是一个"胶水层"——帮你把模型、数据源、工具、记忆等组件粘合在一起。

## 核心概念速查

### Model I/O：与 LLM 交互的基础

LangChain 对各家 LLM API（OpenAI、Anthropic、本地模型等）做了统一封装：

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(model="gpt-4", temperature=0)
response = llm.invoke("你好，请介绍一下自己")
```

关键组件：
- **ChatModel**：统一的 LLM 调用接口，切换模型只需换一行代码
- **PromptTemplate**：可复用的 Prompt 模板，支持变量填充
- **OutputParser**：将 LLM 的自由文本输出解析为结构化数据

### Chain：把步骤串起来

Chain 是 LangChain 最基础的概念——将多个步骤串联成一个处理流水线：

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("用一句话解释什么是 {topic}")
chain = prompt | llm | StrOutputParser()

result = chain.invoke({"topic": "RAG"})
```

使用 LCEL（LangChain Expression Language）语法，用 `|` 管道符号串联各组件，代码简洁直观。

### Retriever：检索的核心抽象

Retriever 是 RAG 系统的关键组件，负责从数据源中检索相关文档：

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

# 创建向量存储
vectorstore = FAISS.from_texts(documents, OpenAIEmbeddings())

# 创建检索器
retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# 检索
docs = retriever.invoke("什么是向量数据库？")
```

Retriever 的设计是高度抽象的——同一个接口可以对接不同的后端（FAISS、Chroma、Pinecone 等），切换向量数据库只需改一行配置。

常见的 Retriever 变体：
- **VectorStoreRetriever**：基于向量相似度检索
- **SelfQueryRetriever**：先从问题中提取元数据过滤条件，再做向量检索
- **EnsembleRetriever**：组合多种检索策略（如 BM25 + 向量）
- **ContextualCompressionRetriever**：检索后对文档进行压缩，只保留与问题相关的部分

### RAG Chain：检索 + 生成的完整链路

把 Retriever 和 LLM 串起来，就是一个完整的 RAG 系统：

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_template("""
基于以下上下文回答问题：
{context}

问题：{question}
""")

rag_chain = (
    {"context": retriever, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("RAG 的分块策略有哪些？")
```

### Agent：让 LLM 自己做决策

Agent 是一种更高级的架构——LLM 根据用户问题动态选择使用哪些工具，而不是按照预设的固定链路执行：

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.tools import tool

@tool
def search(query: str) -> str:
    """搜索相关信息"""
    # 实际调用搜索 API
    return search_results

@tool
def calculator(expression: str) -> float:
    """计算数学表达式"""
    return eval(expression)

llm = ChatOpenAI(model="gpt-4")
tools = [search, calculator]

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools)

result = executor.invoke({"input": "搜索 RAG 的最新论文，并计算 2024 年相关论文的增长比例"})
```

Agent 的核心价值：
- **动态决策**：LLM 根据问题自主判断需要调用哪些工具、以什么顺序调用
- **工具扩展**：通过 `@tool` 装饰器可以快速添加新能力
- **适应复杂任务**：处理需要多步推理和工具组合的复杂问题

### Memory：对话记忆管理

在多轮对话中，需要维护对话历史。LangChain 提供了多种 Memory 实现：

```python
from langchain.memory import ConversationBufferMemory, ConversationSummaryMemory

# 方式 1：保存完整对话历史
buffer_memory = ConversationBufferMemory()

# 方式 2：对历史对话进行摘要，节省 token
summary_memory = ConversationSummaryMemory(llm=llm)
```

| Memory 类型 | 特点 | 适用场景 |
|-------------|------|---------|
| BufferMemory | 保存完整对话 | 短对话、对话轮次少 |
| SummaryMemory | LLM 摘要压缩历史 | 长对话、节省 token |
| WindowMemory | 只保留最近 K 轮 | 中等长度对话 |
| EntityMemory | 提取对话中的实体信息 | 需要跨对话记忆实体 |

## LangGraph：从 Chain 到有状态的图

LangGraph 是 LangChain 团队推出的扩展框架，用**图（Graph）**来编排更复杂的 LLM 工作流：

```python
from langgraph.graph import StateGraph, END

# 定义状态
class AgentState(TypedDict):
    messages: list
    next_action: str

# 定义节点函数
def research_node(state):
    # 执行研究任务
    return {"messages": [...], "next_action": "write"}

def write_node(state):
    # 执行写作任务
    return {"messages": [...]}

# 构建图
graph = StateGraph(AgentState)
graph.add_node("research", research_node)
graph.add_node("write", write_node)
graph.add_edge("research", "write")
graph.add_edge("write", END)
graph.set_entry_point("research")

app = graph.compile()
```

LangGraph 相比纯 Chain 的优势：
- **有状态**：每个节点可以读写共享状态，支持跨节点的信息传递
- **条件分支**：根据状态动态决定下一步走哪个节点
- **循环**：支持迭代执行（如"搜索 → 评估 → 再搜索"的循环）
- **可观测性**：图的执行过程可以可视化，便于调试

## 选型建议

| 场景 | 推荐方案 |
|------|---------|
| 简单问答、文本处理 | Chain（LCEL） |
| RAG 知识库 | Retriever + Chain |
| 需要工具调用的智能助手 | Agent |
| 复杂多步工作流（含循环和分支） | LangGraph |
| 多轮对话 | Chain/Agent + Memory |

## 总结

LangChain 的核心价值在于**标准化和可组合性**——每个组件都有统一的接口，可以自由组合。理解 Chain、Retriever、Agent、Memory 这四个核心概念，就掌握了 LangChain 的大部分使用场景。当工作流复杂度超出 Chain 的线性模型时，LangGraph 提供了更灵活的图编排能力。
