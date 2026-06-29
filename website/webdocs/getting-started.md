# 快速开始

5 分钟跑通你的第一个 agent。

## 安装

```bash
pip install rein-agent
```

装完即可接入真实大模型(100+ 厂商,经 LiteLLM),代码里照常 `import rein`。

需要额外能力,按需装 extras:

| Extras | 装什么 | 何时需要 |
|---|---|---|
| `rein-agent[docker]` | docker SDK | 用 `DockerRuntime` 沙箱执行工具 |
| `rein-agent[otel]` | opentelemetry | 把运行记录导出到 OpenTelemetry |
| `rein-agent[cli]` | typer | 用 `rein new` / `rein dev` 脚手架 |

---

## 不用 key 先跑通(推荐第一步)

Rein 内置 `MockProvider`,不联网、不烧 token,就能跑通完整的多轮工具循环 —— 最适合先感受一下:

```python
from rein import Agent, MockProvider, ToolCall

# 预设一段「剧本」:第 1 次模型要调 add,第 2 次给出最终回答
agent = Agent(provider=MockProvider([
    [ToolCall(id="1", name="add", arguments={"a": 3, "b": 4})],
    "3 加 4 等于 7。",
]))

@agent.tool
def add(a: int, b: int) -> int:
    "计算两个整数之和"
    return a + b

result = agent.run("帮我算 3 加 4")
print(result)            # 3 加 4 等于 7。
print(result.steps)      # 看每一步:model → tool → model
```

`print(result)` 直接给答案;想看过程就取 `.steps` / `.usage` / `.stop_reason` —— 一个对象,两种用法。

---

## 用真实模型(需要 key)

填好对应厂商的环境变量,把 `provider=` 换成 `model=`:

```python
from rein import Agent

agent = Agent("anthropic/claude-opus-4-8")   # 寻址对齐 LiteLLM:厂商/模型

@agent.tool
def now() -> str:
    "返回当前日期"
    return "2026-06-28"

print(agent.run("今天几号?用工具查。"))
```

```bash
export ANTHROPIC_API_KEY=sk-ant-...     # 或 OPENAI_API_KEY / DEEPSEEK_API_KEY ...
python main.py
```

!!! tip "一行切厂商"
    `Agent("anthropic/claude-opus-4-8")` → `Agent("openai/gpt-4o")` → `Agent("deepseek/deepseek-chat")`,业务代码一行不用改。还能 `Agent("anthropic/...", fallback=["openai/gpt-4o"])` 主备自动切换。

---

## 用脚手架起一个项目

```bash
pip install "rein-agent[cli]"
rein new myagent              # 极简起点:一个 main.py + .env.example
rein new mybot --template coder   # 带读文件/跑命令工具 + 审批的雏形

cd myagent
cp .env.example .env          # 填入你的 key
python main.py
```

---

## 下一步

- :material-cube-outline: [核心概念](concepts.md) —— Agent / Session / Chat、状态机、熔断
- :material-tools: [写工具](guides/tools.md) —— 把普通函数变成模型能调的工具
- :material-content-save-outline: [人工审批与断点续跑](guides/resume.md)
- :material-book-open-variant: [设计哲学](design.md) —— 为什么这么薄
