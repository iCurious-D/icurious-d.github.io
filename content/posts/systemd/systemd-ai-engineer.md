+++
date = '2026-08-31T20:00:00+08:00'
draft = false
title = 'systemd II：转向 AI 工程后，我重新理解了服务管理'
tags = ['systemd', 'AI', 'LLM', 'Linux', 'GPU', '工程化']
categories = ['AI 技术笔记']
description = '从 Python 开发转向 AI 工程，重新梳理 systemd 在模型就绪、GPU 服务、cgroup 资源控制、凭据、批处理和发布回滚中的职责与边界。'
+++

2023 年整理第一篇 systemd 笔记时，我关心的是 Python 服务如何稳定运行。到了 2026 年 8 月，从 Python 开发转向 AI 工程，我需要问的问题变了：模型还在加载时能否接流量，一次 worker 扩容会不会复制整份权重，显存耗尽为什么没有触发进程重启？

这些变化没有让 systemd 失去作用，却要求我更准确地划分它的职责。本文承接 [systemd I](/posts/systemd/systemd-python-engineer/)，以单机 Linux 上自管的 AI 服务为背景，使用成熟的 systemd 配置能力，不以某个新版本或特定推理框架为前提。

## 1. AI 服务里的四种生命周期

我会先拆开四类组件，再决定谁需要独立的 unit：

| 组件 | 生命周期特点 | 运行关注点 |
| --- | --- | --- |
| API / RAG 编排层 | 长期运行，主要等待外部 I/O | 请求超时、限流、取消传播 |
| 本地模型推理进程 | 启动慢，占用 GPU 和大量内存 | 模型就绪、显存预算、批处理并发 |
| 文档解析 / embedding worker | 持续消费，有 CPU 和内存峰值 | 背压、任务确认、失败重试 |
| 评测 / 离线索引构建 | 有起点和终点的作业 | 输入版本、产物完整性、退出码 |

并非每个 AI 应用都有本地 GPU。如果只调用远端模型 API，systemd 管理的是本地应用进程；上游限流、额度和模型质量依然需要应用层处理。

我倾向于把独立占用 GPU 的推理进程与 API 层分开。这样 API 的代码发布不必每次重新加载模型，也便于分别分配内存和停止期限。但拆分会增加内部接口、鉴权和可观测性的成本，小应用应按实际需要取舍。

## 2. active、ready 和回答正确是三件事

`Type=exec` 能确认进程执行成功，却不能确认模型已经加载，更不能确认检索和生成结果正确。即使进程和端口都存在，第一次请求仍可能触发很长的初始化。

我把服务状态分成三层：

- **存活：** 进程和事件循环仍能工作。
- **就绪：** 必要配置和本地模型已初始化，能够接受目标请求。
- **质量：** 固定评测集上的回答、引用和安全行为达到发布标准。

readiness 应覆盖接受请求所必需的条件，不能只是返回一个常量；也不宜在每次探测时执行昂贵推理或调用付费模型。可以将预热检查结果缓存为本地状态。liveness 则应尽量避免因为一个外部向量库短暂超时就要求重启整个进程。

有两条常见接入路线：

1. 应用支持 systemd notification 时使用 `Type=notify`，初始化完成后发送 `READY=1`，并为启动配置有限的 `TimeoutStartSec`。
2. 应用没有 notification 支持时，继续用 `Type=exec`，在流量入口和发布脚本中等待 readiness。必要时可用有截止时间的 `ExecStartPost` 探测脚本衔接启动依赖。

只把 `Type` 改成 `notify` 不会自动产生就绪通知，反而会等到超时。对于前台 `exec` 服务，也不能以为设置较大的 `TimeoutStartSec` 就会替你等待权重加载。通知与启动状态的定义见 [systemd.service 手册](https://www.freedesktop.org/software/systemd/man/253/systemd.service.html)。

`After=model.service` 只表达顺序；`Requires=` 和 `Wants=` 表达不同强度的依赖关系。若模型 unit 过早报告启动完成，这些配置无法补出业务就绪判断。`network-online.target` 也不保证远端模型或数据库可用，应用仍然要做有上限的连接重试。

## 3. 一份 AI API 的主机级运行约定

以下示例针对调用独立模型服务的 ASGI API，假设 `aiapp` 账户、项目虚拟环境、`api.main:app` 和所列目录已经按部署流程准备好。它不是 GPU 模型加载器，也不假设某个推理框架的参数。

```ini
# /etc/systemd/system/ai-api.service
[Unit]
Description=AI Application API
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=300
StartLimitBurst=3

[Service]
Type=exec
User=aiapp
Group=aiapp
WorkingDirectory=/opt/ai-api/current
Environment=PYTHONUNBUFFERED=1
EnvironmentFile=/etc/ai-api/runtime.env
ExecStart=/opt/ai-api/current/.venv/bin/python -m uvicorn api.main:app --host 127.0.0.1 --port 8080 --workers 1 --timeout-graceful-shutdown 60
Restart=on-failure
RestartSec=15s
TimeoutStopSec=90s
KillMode=control-group
MemoryAccounting=true
MemoryHigh=6G
MemoryMax=8G
CPUQuota=200%
TasksMax=256
StateDirectory=ai-api
CacheDirectory=ai-api
Environment=HF_HOME=/var/cache/ai-api/huggingface
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/ai-api /var/cache/ai-api
NoNewPrivileges=true
PrivateTmp=true
UMask=0077
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

这些数值是解释配置的起点，不是容量建议。上线前要结合依赖版本、请求长度、并发和机器资源测量；启动命令中的 Uvicorn 参数也应对照已安装版本的 `--help`。

与普通 API 相比，我尤其关注以下边界：

- `MemoryHigh` 对应内存压力下的节流和回收机制；`MemoryMax` 是最后的硬边界，可能导致 cgroup 内 OOM。它们不是“性能调优后一定更快”的开关。
- `CPUQuota=200%` 表示约两个 CPU 的时间配额，不是绑定两个指定核心，也不是整机 CPU 的 200%。
- `TasksMax` 限制任务数量，线程也在其中。数值太小可能让数值计算库或分词器创建线程失败。
- 资源限制按层级生效，父 slice 的约束也需要检查。

这些配置依赖内核、cgroup 和控制器支持，建议在 cgroup v2 环境验收，而不是看到 unit 被接受就认为限制已完整生效。资源语义参考 [systemd.resource-control 手册](https://www.freedesktop.org/software/systemd/man/257/systemd.resource-control.html)。

可写目录必须和应用实际配置一致；`StateDirectory` 创建的是目录，不会自动让业务把数据库或索引写进去。`ProtectSystem=strict` 也不能直接套用到所有机器学习项目，需要明确 tokenizer、编译缓存和下载目录。样例显式设置了 Hugging Face 缓存位置，其他库仍需分别检查。

## 4. GPU 不在普通内存配额里

这是我从 Python 服务思维转过来后最需要纠正的一点：`MemoryMax=8G` 约束的是主机内存，不是 CUDA 显存。进程完全可能在主机内存使用平稳时抛出 CUDA OOM。

显存预算需要由模型服务配置和实测共同确定：权重、KV cache、上下文长度、并发序列数、临时张量都会影响峰值。减少上下文或并发，与限制 Linux 进程内存，是不同的控制手段。

如果模型在每个应用 worker 中初始化，照搬普通 Web 服务的多 worker 经验可能复制多份模型。应先确认推理框架的进程模型和权重共享方式，再决定扩容位置。

```ini
# 独立推理服务中的一种设备选择示例
Environment=CUDA_VISIBLE_DEVICES=0
```

这个变量用于 CUDA 可见设备选择，不能充当多租户安全隔离，也不会阻止另一服务选择同一张 GPU。设备序号还要结合部署环境确认；需要稳定身份时可根据 CUDA 支持情况使用 GPU UUID。设备访问控制、驱动权限以及 MIG 等机制属于另一层问题。

我不会随手给 GPU unit 加 `PrivateDevices=true`，因为隐藏设备可能让驱动无法工作。也不会给出一个只放行 `/dev/nvidia0` 就声称完整可用的配置：实际所需设备节点、驱动和容器运行环境需要在目标机器验证。

## 5. 自动恢复不能掩盖容量与质量问题

AI 服务里至少要区分下面几类故障：

| 现象 | 应优先采取的动作 |
| --- | --- |
| 启动时权重文件不存在 | 修复模型路径或制品，避免持续重启 |
| CUDA OOM 被框架捕获，进程仍在 | 降低负载并确认进程是否可恢复，不能只等 Restart |
| 主机内存超限导致退出 | 查 cgroup 与内核记录，再核对内存预算 |
| 远端模型返回限流 | 应用层退避、限流或降级 |
| HTTP 正常但回答质量退化 | 评测定位并回滚模型、Prompt 或索引版本 |

`Restart=on-failure` 只对符合条件的服务失败做反应，不理解这些业务语义。无上限的任务重试加上进程重启，可能放大外部调用成本。需要将进程恢复、任务重试次数和业务幂等分别设计。

如果启用 `WatchdogSec`，应用必须按协议定期发送 watchdog 通知，而且这个信号应反映关键路径仍有进展。单独开个永远能发心跳的线程，可能掩盖主工作线程已经卡死的事实。没有实现协议前，不要为了“更可靠”直接添加该配置。

## 6. 停止服务，是对在途请求作出承诺

LLM 流式回答和 embedding 批次都可能比普通 HTTP 请求长。我希望停止顺序是：先从流量入口摘除实例，等待传播，停止接受新任务，再在期限内完成或取消在途工作，最后退出进程。

示例给 Uvicorn 60 秒的优雅退出时间，给 systemd 90 秒作为外层收尾预算。两者的差额留给框架清理等步骤；实际数值应来自服务的超时约定。如果框架不传播取消，客户端断开后远端推理可能仍继续运行和计费。

队列 worker 则需要处理租约、确认时机与重复执行。索引构建不要直接覆盖线上目录：先写入新版本，校验向量数、元数据和检索结果，再切换读入口。即使最终被强制结束，也应留下可清理的中间产物，而不是半份“成功”的索引。

## 7. 密钥、日志与可追溯发布

第一篇里的环境文件适合普通运行配置。对于模型 API key，可以在本机 systemd 支持时使用：

```ini
LoadCredential=model_api_key:/etc/ai-api/secrets/model_api_key
```

应用从 `CREDENTIALS_DIRECTORY` 指向的目录读取 `model_api_key`，需要自己实现读取逻辑。它不会自动转换成 SDK 所需的环境变量。部署端源文件仍需控制权限；普通 `LoadCredential` 不等于源文件已经加密。执行环境、沙箱与凭据机制参考 [systemd.exec 手册](https://www.freedesktop.org/software/systemd/man/257/systemd.exec.html)。

日志应包含 request ID、模型版本、Prompt 版本、索引版本、耗时和错误类型；不要默认记录完整用户输入、检索原文或密钥。journal 能帮我定位进程事件，但请求链路追踪、指标、费用和质量评测仍需要专门的应用观测。

发布时，我希望代码、依赖和模型制品都有明确版本。`/opt/ai-api/current` 只是一个指向版本目录的入口：先准备并验证新版本，再切换入口、重启、等待 readiness 和冒烟检查。失败时切回旧入口并重新启动；这不自动解决数据库迁移或索引格式不兼容。

单个实例执行 `restart` 会有服务空窗。需要连续可用时，应准备第二个实例或蓝绿端口，通过反向代理切换流量，并确认机器有能力同时容纳两份模型。systemd 本身不会自动完成这套流量切换。

## 8. 我会怎样验收一份 AI 服务 unit

```bash
sudo systemd-analyze verify /etc/systemd/system/ai-api.service
sudo systemctl daemon-reload
sudo systemctl enable --now ai-api.service
systemctl status ai-api.service --no-pager
systemctl show ai-api.service -p Result -p NRestarts -p MemoryCurrent -p ControlGroup
journalctl -u ai-api.service -b -n 100 --no-pager
systemd-cgtop
```

这些命令只是入口，完整验收还需要在测试环境覆盖：

1. 延长模型加载时间，确认就绪前不会接业务流量。
2. 制造可控的超时、限流和错误退出，确认恢复行为符合预期。
3. 发起长流式请求后停止实例，确认摘流量、超时和取消路径。
4. 测量目标并发与上下文下的主机内存和 GPU 显存峰值。
5. 中断索引构建，再重跑同一任务，确认不会污染线上版本。
6. 切换新版本失败后回滚，检查代码、模型和索引是否匹配。

本文没有提供未经实测的吞吐量或可用性数字；这些检查项需要在实际硬件和部署版本上执行后才能成为验证结论。

对于单机、固定 GPU 服务器或小规模内网应用，systemd 能让运行约定保持清楚。需要跨主机调度、自动扩缩容和副本协调时，再交给合适的编排系统；若已经由容器编排管理应用生命周期，就应避免再嵌套一套重复的应用守护逻辑。

从 Python 开发转向 AI 工程后，我对 systemd 的要求更具体了：知道它如何管理进程，也知道哪些状态必须由应用告诉它。服务能重启只是起点；模型何时就绪、资源能承受多少请求、任务如何恢复、产物如何回滚，才构成一份完整的运行设计。
