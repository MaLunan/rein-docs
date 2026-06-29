# 脚手架

`rein new` / `rein dev` 背后的纯函数。教程见 [脚手架 CLI](../guides/cli.md)。

---

## create_project

```python
def create_project(name: str, template: str = "minimal", target_dir: str | Path = ".") -> Path
```

生成一个极简可跑的 agent 项目,返回项目目录 `Path`。这是 `rein new` 的核心 —— 一个**纯函数**(不依赖 typer),所以可单测、也可在代码里直接调。

| 参数 | 说明 |
|---|---|
| `name` | 项目名(同时作为目录名) |
| `template` | `"minimal"` 或 `"coder"` |
| `target_dir` | 在哪个目录下创建(默认当前目录) |

**抛**:`ValueError`(模板不存在 / 项目名非法)、`FileExistsError`(目标已存在,不覆盖)。

生成的文件(**不堆空目录**):`main.py` + `.env.example` + `README.md` + `rein.toml`。

```python
from rein import create_project
proj = create_project("myagent", template="coder")   # → Path("./myagent")
```

---

## available_templates

```python
def available_templates() -> list[str]
```

列出可用模板名。

```python
from rein import available_templates
available_templates()   # ['minimal', 'coder']
```

| 模板 | 内容 |
|---|---|
| `minimal` | 5 行示例(一个工具 + run) |
| `coder` | `read_file` / `run_shell` 工具雏形 + `permission="ask"` |

---

## CLI 命令

装 `pip install "rein-agent[cli]"` 后:

```bash
rein new <name> [--template minimal|coder]   # 生成项目
rein dev [main.py]                           # 监听文件、热重载
```

`rein dev` 用标准库轮询 mtime 重启子进程,并给子进程设 `REIN_DEV=1`(你可据此挂调试追踪)。刻意不做 `rein run` —— 跑项目就是 `python main.py`。
