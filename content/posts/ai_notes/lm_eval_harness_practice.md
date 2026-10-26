+++
date = '2026-10-05T14:00:00+08:00'
draft = false
title = '模型评测实战：用 lm-evaluation-harness 跑通 Benchmark'
tags = ["AI", "评测", "benchmark"]
categories = ['AI 技术笔记']
description = "EleutherAI 的 lm-evaluation-harness 是模型层评测的事实标准：安装与基础命令、loglikelihood 与 generate 两种任务类型、MMLU/GSM8K/C-Eval 指标解读，以及微调前后如何用它验证模型能力变化。"
+++

## 模型层评测为什么需要标准工具

上一篇文章讲了应用层评估（见[LLM 应用评估体系](/posts/ai_notes/llm_evaluation_system/)），本文补上**模型层评测**：你的模型本身在通用能力上是什么水平。

自己写评测脚本会遇到一堆坑：数据集下载、few-shot 构造、prompt 模板、指标口径。**lm-evaluation-harness**（EleutherAI 出品）把这些全部标准化了，是目前开源社区评测模型的事实标准，也是很多论文报告分数的默认工具。

## 安装与基础用法

```bash
pip install lm-eval

# 一条命令跑通 MMLU 评测
lm_eval --model hf \
    --model_args pretrained=Qwen/Qwen2.5-7B-Instruct,dtype=bfloat16 \
    --tasks mmlu \
    --num_fewshot 5 \
    --batch_size auto \
    --output_path ./results
```

参数说明：

| 参数 | 含义 |
|------|------|
| `--model hf` | 通过 Hugging Face Transformers 加载模型 |
| `--model_args` | 模型加载参数：路径、精度、并行策略 |
| `--tasks` | 评测任务，可逗号分隔多个 |
| `--num_fewshot` | few-shot 示例数，不同 benchmark 有约定俗成的值 |
| `--batch_size auto` | 自动批大小，显存不足会自动调小 |
| `--output_path` | 结果 JSON 输出目录 |

## 两种任务类型：loglikelihood vs generate

理解任务类型才能看懂结果：

| 类型 | 原理 | 适用 benchmark |
|------|------|---------------|
| loglikelihood | 模型给每个选项算条件概率，取概率最高者 | 选择题（MMLU、C-Eval） |
| generate | 模型自由生成答案，再用规则/正则判对错 | 数学题（GSM8K）、代码题 |

以 MMLU（选择题）为例：把题目和 4 个选项给模型，模型分别计算「A/B/C/D」作为下一个 token 的概率，选概率最高的。指标是 **acc**（看首 token 概率）和 **acc_norm**（按选项文本长度归一化后的概率）——**acc_norm 是更公平的指标**，因为长选项天然概率偏低。

## 常用 Benchmark 与指标解读

| Benchmark | 考察能力 | 任务类型 | 主要指标 |
|-----------|---------|---------|---------|
| MMLU | 多学科知识（57 类） | loglikelihood | acc, acc_norm |
| C-Eval | 中文综合知识 | loglikelihood | acc, acc_norm |
| GSM8K | 小学数学应用题 | generate | exact_match |
| BBH | 复杂推理 | generate | exact_match |
| HumanEval | 代码生成 | generate | pass@1 |

看结果时注意口径：**不同 benchmark 的分数不可直接对比**，只有同任务同 few-shot 设置下的分数才有可比性。论文里「MMLU 5-shot acc」这种表述就是在声明口径。

## 实战：微调前后对比

结合 LoRA 微调流程（见[大模型微调实战](/posts/ai_notes/lora_finetuning_practice/)），评测的正确姿势是**微调前后跑同一套任务**：

```bash
# 1. 微调前基线
lm_eval --model hf \
    --model_args pretrained=Qwen/Qwen2.5-7B-Instruct \
    --tasks ceval-valid,gsm8k \
    --output_path ./results/baseline

# 2. 微调后
lm_eval --model hf \
    --model_args pretrained=./output/lora-merged \
    --tasks ceval-valid,gsm8k \
    --output_path ./results/finetuned

# 3. 对比两次的 results.json 中 metric 字段
```

三个实战提醒：

1. **通用能力会退化**：领域微调后 MMLU 类分数可能下降，这是正常现象，重点看目标领域指标是否提升
2. **评测是抽样的**：分数有随机性，差 1~2 分不一定是真差距，必要时跑多次取平均
3. **显存紧张用量化加载**：`dtype=bfloat16` 换成 `load_in_4bit=True`（需 bitsandbytes），评测速度慢但能跑

## 评测自己的数据：自定义 Task

harness 支持自定义评测集，适合测领域能力：

```bash
# 用 --include_path 指向自己的 task 目录
lm_eval --model hf \
    --model_args pretrained=./output/lora-merged \
    --tasks my_domain_qa \
    --include_path ./my_tasks \
    --output_path ./results
```

自定义 task 目录结构（YAML 配置 + 数据集文件）：

```yaml
# my_tasks/my_domain_qa.yaml
task: my_domain_qa
dataset_path: json
dataset_kwargs:
  data_files: ./my_tasks/domain_qa.jsonl
output_type: generate_until
num_fewshot: 0
metric_list:
  - metric: exact_match
```

数据集是 JSONL，每行 `{"question": "...", "answer": "..."}`。这让领域评测和通用 benchmark 使用同一套流程，结果可直接对比。

## 面试要点速记

- 模型层评测（harness/benchmark）与应用层评估（RAGAS/Judge）是两层，缺一不可
- loglikelihood（选择题）vs generate（生成题）两种任务类型
- MMLU 看 acc_norm，比 acc 更公平
- 分数口径 = 任务 + few-shot 数 + 模型精度，三者都要说清楚
- 微调后通用能力可能退化，重点看目标领域指标

## 延伸阅读

- [LLM 应用评估体系：从评测集设计到 LLM-as-a-Judge](/posts/ai_notes/llm_evaluation_system/)
- [大模型微调实战：LoRA 与 QLoRA 的原理、流程与踩坑](/posts/ai_notes/lora_finetuning_practice/)
- [DeepSeek 模型应用实战](/posts/ai_notes/deepseek_application_practice/)
