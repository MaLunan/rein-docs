# 脚手架(CLI)

一条命令起一个能直接跑的项目。

```bash
pip install "rein-agent[cli]"
```

## rein new

```bash
rein new myagent                  # minimal 模板
rein new mybot --template coder   # coder 模板(读文件/跑命令 + 审批)
```

生成的项目**极简** —— 就这几个文件,不堆空目录:

```
myagent/
├── main.py          # 能直接跑的起点
├── .env.example     # key 模板
├── README.md
└── rein.toml        # 可选配置示例
```

跑起来:

```bash
cd myagent
cp .env.example .env      # 填入你的 key
python main.py
```

!!! note "刻意不做 `rein run`"
    跑项目就是 `python main.py`,不搞多余命令。

## 两个模板

| 模板 | 内容 |
|---|---|
| `minimal` | 5 行示例(一个工具 + run) |
| `coder` | `read_file` / `run_shell` 工具雏形 + `permission="ask"`(危险操作要批准) |

## rein dev(热重载)

```bash
rein dev          # 监听 main.py,改了就自动重启
```

基于文件 mtime 轮询 + 重启子进程(不引入额外依赖)。会给子进程设 `REIN_DEV=1`,你可以据此挂调试追踪:

```python
import os
if os.getenv("REIN_DEV"):
    agent.on("step", lambda c: print("step:", c.stage.value))
```

## 编程式生成

脚手架的核心是个纯函数,也能在代码里调用:

```python
from rein import create_project, available_templates

available_templates()                         # ['minimal', 'coder']
create_project("myagent", template="coder")   # 返回项目目录 Path
```
