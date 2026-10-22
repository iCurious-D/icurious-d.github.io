+++
date = '2026-10-05T17:10:00+08:00'
draft = false
title = 'AI 应用工程学习路线'
description = '按 RAG、Agent、模型与评测、工程化四条主线整理 AI 应用工程文章，形成从原理到生产实践的阅读路径。'
+++

这里按能力而不是发布日期组织文章。推荐先理解系统边界，再阅读实现细节和评测方法。

## RAG：从检索到可信回答

1. [RAG 系统设计与优化实践](/posts/ai_notes/rag_design_and_optimization/)
2. [RAG 优化实战：从能跑到能用](/posts/ai_notes/rag_optimization_in_practice/)
3. [向量数据库选型与实践](/posts/ai_notes/vector_db_comparisons/)
4. [GraphRAG 与知识图谱增强检索](/posts/ai_notes/graphrag_practice/)
5. [LLM 应用评估体系](/posts/ai_notes/llm_evaluation_system/)

## Agent：从工具调用到工作流

1. [Function Calling 实战](/posts/ai_notes/function_calling_practice/)
2. [LangGraph Agent 工作流实战](/posts/ai_notes/agent_workflow_langgraph/)
3. [MCP 协议入门与实战](/posts/ai_notes/mcp_intro_practice/)
4. [Codex 编程 Agent 实战](/posts/ai_notes/codex_coding_agent/)

## 模型与评测

1. [Transformer 面试知识梳理](/posts/ai_notes/transformer_interview_notes/)
2. [LoRA / QLoRA 微调实践](/posts/ai_notes/lora_finetuning_practice/)
3. [用 lm-evaluation-harness 跑通 Benchmark](/posts/ai_notes/lm_eval_harness_practice/)
4. [DeepSeek 模型应用实战](/posts/ai_notes/deepseek_application_practice/)

## 工程化

1. [Prompt Engineering 实战指南](/posts/ai_notes/prompt_engineering_guide/)
2. [LangChain 核心概念](/posts/ai_notes/langchain_core_concepts/)
3. [LLM 应用服务生产化](/posts/ai_notes/llm_service_production/)
4. [systemd I：Python 工程师如何把程序交给 Linux 管理](/posts/systemd/systemd-python-engineer/)
5. [systemd II：转向 AI 工程后，我重新理解了服务管理](/posts/systemd/systemd-ai-engineer/)

核心文章尽量遵循统一结构：问题背景、Baseline、方案选型、核心实现、验证方法、失败案例、适用边界和面试追问。项目页负责提供实践证据，文章负责解释可复用的方法。
