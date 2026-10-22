+++
date = '2023-08-31T20:00:00+08:00'
draft = false
title = 'systemd I：Python 工程师如何把程序交给 Linux 管理'
tags = ['systemd', 'Python', 'Linux', '部署', '工程化']
categories = ['后端工程']
description = '从 Python 工程师的部署需求出发，理解 systemd unit、虚拟环境、日志、重启、优雅退出和 timer，完成从脚本到 Linux 服务的第一步。'
+++

写完一个 Python 程序之后，我最初关心的是它能不能跑起来。真正把程序放到 Linux 服务器上，问题就变成了：SSH 断开后谁负责它，机器重启后谁启动它，半夜退出后谁能告诉我原因？

这篇笔记整理于 2023 年 8 月，面向部署 Python API、后台 worker 和定时脚本的开发者。示例使用当时已经存在的 systemd 能力，不依赖某个最新发行版。命令在以 systemd 为服务管理器的 Linux 主机执行；先用 `systemctl --version` 查看本机版本。

## 1. 从进程想到服务

`nohup python app.py &` 能让程序脱离终端，但没有把运行目录、身份、失败恢复和开机启动组织成一个可检查的约定。systemd 的价值，是把这些约定写进 unit 文件，由服务管理器维护进程的生命周期。

先认识三个对象就足够开始工作：

| 对象 | 我的使用场景 |
| --- | --- |
| `.service` | 常驻 API、队列消费者，或者执行一次就退出的脚本 |
| `.timer` | 按时间触发某个 service |
| `.target` | 表达一组 unit 或某个系统运行阶段 |

一个 service 不一定只包含一个进程。主进程启动的子进程通常也处于该服务的 cgroup 中，停止服务时应一起考虑。这对 Gunicorn 的 master/worker、Python multiprocessing 尤其重要。

## 2. 把运行环境写清楚

假设项目已经部署为下面的布局：

```text
/opt/order-api/
├── .venv/
└── app.py
/etc/order-api/order-api.env
/var/lib/order-api/
```

其中 `.venv` 已安装 Flask 和 Gunicorn，`app.py` 暴露 Flask 对象 `app`，服务账户 `orderapi` 已创建。代码和虚拟环境由部署账户维护，服务账户只需要读和执行；业务写入放到 `/var/lib/order-api`。下面的配置不会自动替你安装依赖或创建账户。

创建 `/etc/systemd/system/order-api.service`：

```ini
[Unit]
Description=Python Order API
After=network.target
StartLimitIntervalSec=60
StartLimitBurst=5

[Service]
Type=exec
User=orderapi
Group=orderapi
WorkingDirectory=/opt/order-api
Environment=PYTHONUNBUFFERED=1
EnvironmentFile=/etc/order-api/order-api.env
ExecStart=/opt/order-api/.venv/bin/gunicorn --bind 127.0.0.1:8000 --workers 2 --access-logfile - --error-logfile - app:app
Restart=on-failure
RestartSec=5s
TimeoutStopSec=45s
KillMode=control-group
StandardOutput=journal
StandardError=journal
SyslogIdentifier=order-api
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true
StateDirectory=order-api

[Install]
WantedBy=multi-user.target
```

这里的 worker 数量只是演示，实际要结合 CPU、内存和负载测量。端口绑定在回环地址，适合由同机反向代理转发；它本身不提供公网入口、TLS 或认证。

我会优先检查以下几个容易误解的地方：

- **直接调用虚拟环境里的程序。** 不需要 `source .venv/bin/activate`。systemd 不会自动读取交互终端里的 `.bashrc`，也不继承你手工激活的环境。
- **明确工作目录。** 相对路径读取失败，经常是因为程序默认把当前目录当成项目目录。
- **使用独立账户。** API 监听 8000 端口并不需要 root；文件权限应该围绕实际读写需求分配。
- **以前台方式运行。** 不加 Gunicorn 的 daemon 参数，不在命令末尾加 `&`。让 systemd 直接管理前台主进程。

`Type=exec` 会在执行程序这一步成功后继续启动流程，能更直接地暴露可执行文件不存在等错误；这仍然不代表 API 已经能够处理请求。旧环境如果不支持它，可以改用 `Type=simple`，并理解其启动成功的判定更早。具体语义见 [systemd.service 官方手册](https://www.freedesktop.org/software/systemd/man/253/systemd.service.html)。

`ExecStart` 也不是普通 shell 命令行。不能期待 `>`、`|`、`&&` 自动生效。确实需要复杂准备逻辑时，我更愿意写一个可以独立测试的启动脚本，最后用 `exec` 替换 shell，避免平白增加一层信号转发。

环境文件使用 `KEY=value`，不要写 `export`，也不要依赖 shell 命令替换。上面的文件没有加 `-` 前缀，缺失会导致启动失败，这适合必需配置。凭据文件应严格限制权限，日志中也不要打印连接串；环境变量不能当作可靠的密钥隔离方案。

## 3. 安装、启动和更新是三件事

保存 unit 后执行：

```bash
sudo systemd-analyze verify /etc/systemd/system/order-api.service
sudo systemctl daemon-reload
sudo systemctl enable --now order-api.service
systemctl status order-api.service --no-pager
curl --fail http://127.0.0.1:8000/health
```

最后一条需要应用自己实现 `/health`，不能把 HTTP 健康检查当成 systemd 自动提供的功能。

`daemon-reload` 让管理器重新加载 unit 定义，不会自动重启应用；`enable` 建立开机启动关系；`start` 启动当前服务。这里用 `enable --now` 同时完成后两件事。

代码或者环境文件更新后，通常需要 `restart` 才会进入新进程。只有应用支持重新加载，并且 unit 配置了相应行为，`reload` 才有实际意义，不能把它当成通用热更新命令。

## 4. 先找到退出原因，再决定怎么重启

```bash
journalctl -u order-api.service -b -n 100 --no-pager
journalctl -u order-api.service --since '30 minutes ago'
journalctl -u order-api.service -f
systemctl show order-api.service -p Result -p ExecMainStatus -p NRestarts
systemctl cat order-api.service
```

Python 标准输出的缓冲会让日志看起来“丢了”，所以示例显式设置 `PYTHONUNBUFFERED=1`。应用仍需要合理的日志等级、异常堆栈和请求标识。journal 的保留时间、磁盘配额和跨重启持久化取决于 journald 配置，不能假设历史日志永远存在。

遇到启动错误，我按这个顺序排查：执行路径是否存在，账户是否能访问工作目录，环境文件是否可读，依赖是否安装在同一个虚拟环境，端口是否被占用，最后才看业务异常。`203/EXEC` 和 `200/CHDIR` 这类状态提示，往往比重复运行启动命令有用。

`Restart=on-failure` 可以恢复部分意外退出，却不会修复错误配置。`RestartSec` 避免立即重试，启动频率限制则给连续失败设一道闸门。触发限制后，先修复原因，再执行：

```bash
sudo systemctl reset-failed order-api.service
sudo systemctl start order-api.service
```

手动 `stop` 不会因为配置了失败重启就立刻被拉起。另一方面，进程还活着但请求已经卡死时，普通重启策略并不能发现问题，需要应用健康检查或额外监控。

## 5. Python 程序要配合停止流程

systemd 停止服务时通常先发送 SIGTERM，超过停止期限后可能强制结束。Gunicorn 等服务器有自己的退出流程；自写 worker 则需要明确处理信号：

```python
import logging
import signal
import threading

logging.basicConfig(level=logging.INFO)
stopping = threading.Event()

def request_stop(signum, frame):
    stopping.set()

signal.signal(signal.SIGTERM, request_stop)
signal.signal(signal.SIGINT, request_stop)

try:
    while not stopping.wait(timeout=1):
        # 在这里领取并处理一项有超时上限的任务
        logging.info("worker tick")
finally:
    logging.info("worker stopped; close clients here")
```

这是信号处理骨架，不是完整队列消费者。实际 worker 应停止领取新任务，给正在处理的任务设置期限，并在完成后确认消息。不能在任务刚开始时就确认，也不能指望 `finally` 在 SIGKILL 或断电时执行。任务幂等和消息重投，仍然由业务设计保证。

`TimeoutStopSec` 要覆盖合理的收尾时间；如果某次网络调用可以无限阻塞，再长的停止期限也解决不了根因。

## 6. 定时 Python 脚本：service 加 timer

例如每天生成报表，先建立 `/etc/systemd/system/order-report.service`：

```ini
[Unit]
Description=Generate daily order report

[Service]
Type=oneshot
User=orderapi
Group=orderapi
WorkingDirectory=/opt/order-api
EnvironmentFile=/etc/order-api/order-api.env
ExecStart=/opt/order-api/.venv/bin/python -m jobs.daily_report
TimeoutStartSec=30min
```

这里假设项目提供了 `jobs.daily_report` 模块。然后建立同名 timer：

```ini
[Unit]
Description=Run order report every day

[Timer]
OnCalendar=*-*-* 02:00:00 Asia/Shanghai
Persistent=true
RandomizedDelaySec=5min

[Install]
WantedBy=timers.target
```

```bash
systemd-analyze calendar '*-*-* 02:00:00 Asia/Shanghai'
sudo systemctl daemon-reload
sudo systemctl enable --now order-report.timer
systemctl list-timers --all
```

`Persistent=true` 能在日历任务因关机等原因错过后补触发，但不会把每个遗漏的业务日期逐一重放。随机延迟可以分散同一时间启动的任务。同一个 service 还处于运行状态时，timer 不会再启动一个相同实例；跨主机并发或人工重复执行仍需业务锁。相关细节见 [systemd.timer 官方手册](https://www.freedesktop.org/software/systemd/man/253/systemd.timer.html)。

## 7. 到什么程度才算掌握

对我而言，第一阶段的目标不是背熟所有配置项，而是能完成一次可解释的部署：

1. 以最小权限账户启动服务，并准确说明解释器、工作目录和配置从哪里来。
2. 重启主机后能自动恢复，失败时能从日志找到原因。
3. 停止服务时能处理在途任务，而不是靠强杀碰运气。
4. 能区分程序退出、启动失败和进程存活但业务不可用。
5. 能把定时任务的触发机制与业务补偿逻辑分开设计。

上线前应在测试机验证：故意写错路径是否能看出原因，异常退出是否按策略恢复，手动停止是否保持停止，重启主机是否恢复，以及报表重复执行会不会产生重复数据。这些是验收方法，不代表本文示例已经在某台生产机上实测。

systemd 帮我把“怎么启动这个 Python 文件”变成了一份可以审查、追踪和复现的运行约定。后续即使部署方式变化，这套关于身份、环境、信号和日志的认识依然有用。

系列下一篇：[systemd II：转向 AI 工程后，我重新理解了服务管理](/posts/systemd/systemd-ai-engineer/)。
