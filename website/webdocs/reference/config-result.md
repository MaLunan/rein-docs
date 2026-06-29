# 配置 · 状态 · 结果

`LoopConfig`(怎么跑)、`Session`(状态快照)、`RunResult`(跑完的报告)。

---

## `LoopConfig`

Agentic loop 的运行配置(熔断 + 权限)。它是 Agent 蓝图的一部分,**不进**可序列化的 Session。

```python
LoopConfig(
    max_iterations: int = 50,
    max_tokens: int | None = 200_000,
    timeout_s: float | None = 120,
    detect_loops: bool = True,
    repeat_threshold: int = 3,
    permission: Literal["allow", "ask", "deny"] = "allow",
)
```

| 字段 | 默认 | 说明 |
|---|---|---|
| `max_iterations` | `50` | **① 轮数闸**。一轮 = 一次「调模型 → 执行工具」。 |
| `max_tokens` | `200_000` | **② 成本闸**。累计(输入+输出)token 上限;`None` 不限。生产头号事故的兜底。 |
| `timeout_s` | `120` | **③ 墙钟闸**(秒),从 run 开始计时;`None` 不限。 |
| `detect_loops` | `True` | **④ 重复闸**开关:是否检测「连续调用完全相同的工具」。 |
| `repeat_threshold` | `3` | 连续多少轮相同调用就判定卡死(仅 `detect_loops=True` 时生效)。 |
| `permission` | `"allow"` | 工具执行前的放行策略,见下。 |

**permission 三种**

- `"allow"` — 直接放行(默认,保证「5 行示例」一路跑完)。
- `"deny"` — 直接拒绝,以错误结果回填,让模型自己应对。
- `"ask"` — 执行前暂停、产出 `need_approval` 中断,等 `resume` —— 用于人工审批(HITL)。

```python
from rein import Agent, LoopConfig

agent = Agent("anthropic/...", config=LoopConfig(
    max_tokens=50_000,      # 把成本闸收紧
    permission="ask",       # 危险操作要审批
))
```

---

## `RunResult`

agent 跑完一次的「结果报告」。**一个对象,两种用法**:`print` 直接看答案,或取字段看细节。

```python
RunResult:
    status: Literal["done", "interrupted"]
    output: str | None
    session: Session
    usage: Usage
    elapsed_s: float | None
    stop_reason: str
    steps: list[Step]
    interrupt: Interrupt | None
```

| 字段 | 说明 |
|---|---|
| `status` | `"done"`=正常跑完;`"interrupted"`=中途中断、等待 resume |
| `output` | 最终的助手回答文本(没有则 `None`) |
| `session` | 完整会话状态。**可序列化存档、可 resume** |
| `usage` | 本次总 token / 成本(`Usage`) |
| `elapsed_s` | 总墙钟耗时(秒) |
| `stop_reason` | `done` / `max_iterations` / `max_tokens` / `timeout` / `loop_detected` / `interrupted` |
| `steps` | 逐步流水账(见 [`Step`](#step)),可回放 |
| `interrupt` | 若 `status="interrupted"`,这里是中断详情(见 [`Interrupt`](#interrupt)) |

`__str__` 返回 `output` —— 所以 `print(result)` 直接显示答案。

```python
result = agent.run("...")
print(result)                          # = result.output
if result.status == "interrupted":
    print(result.interrupt.message)
for s in result.steps:
    print(s.index, s.kind, s.summary)
```

---

## `Step`

运行过程中的「一步」记录 —— 要么调了一次模型,要么执行了一个工具。这是「可观测」的最小形态。

```python
Step:
    index: int
    kind: Literal["model", "tool"]
    summary: str
    tool_calls: list[ToolCall] | None
    is_error: bool | None
    duration_s: float | None
    usage: Usage | None
```

| 字段 | 说明 |
|---|---|
| `index` | 第几步(从 0 开始) |
| `kind` | `"model"`=调了一次模型;`"tool"`=执行了一个工具 |
| `summary` | 简短摘要(模型步是回答截断;工具步是「工具名 → 结果摘要」) |
| `tool_calls` | 仅模型步且要调工具时:这步请求了哪些工具 |
| `is_error` | 仅工具步:是否执行出错 |
| `duration_s` | 这步耗时(秒) |
| `usage` | 这步的 token 用量(主要模型步有) |

---

## `Interrupt`

中断态 —— agent 在工具执行前需要「外部介入」时产生。全部可序列化。

```python
Interrupt:
    type: Literal["need_approval", "need_input", "error"]
    tool_call: ToolCall | None
    message: str
```

| `type` | 含义 |
|---|---|
| `need_approval` | 有工具待人批准(`permission="ask"`) |
| `need_input` | 需要人补充信息(用 `resume(answer=...)` 注入) |
| `error` | 出现可重试错误,调用方可 `resume` 重试或放弃 |

```python
r = agent.run("...")
if r.status == "interrupted":
    itr = r.interrupt
    print(itr.type, itr.message, itr.tool_call)
```

---

## `Session`

一次 agent 运行的**全部状态**(可序列化的状态快照)。设计铁律:只有数据字段,没有业务方法 —— 所有推进逻辑在 loop 里。

```python
Session:
    messages: list[Message]
    stage: Stage
    pending_tool_calls: list[ToolCall]
    iteration: int
    usage: Usage
    repeat_count: int
    last_signature: str | None
    done: bool
    stop_reason: str | None
```

| 字段 | 说明 |
|---|---|
| `messages` | 对话历史(系统/用户/模型/工具,按时间排列) |
| `stage` | 当前状态机阶段(见 [`Stage`](#stage)) |
| `pending_tool_calls` | 待执行的工具调用队列(中断点正在「工具执行前」,这些要随 Session 一起存盘) |
| `iteration` | 已完成轮数(熔断①用) |
| `usage` | 累计 token/成本(熔断②用) |
| `repeat_count` / `last_signature` | 重复检测状态(熔断④用) |
| `done` / `stop_reason` | 是否结束 + 结束原因 |

**序列化往返**(存盘/恢复的地基):

```python
raw = session.model_dump_json()             # 存盘
session2 = Session.model_validate_json(raw) # 读回 → 可喂给 resume
```

---

## `Stage`

状态机的三个阶段(继承 `str`,可直接序列化)。

```python
Stage.CALL_MODEL   # "call_model" 调模型
Stage.RUN_TOOLS    # "run_tools"  执行工具
Stage.DONE         # "done"       结束
```

流转:

```
CALL_MODEL ──(模型给文本)──→ DONE
CALL_MODEL ──(模型要调工具)─→ RUN_TOOLS
RUN_TOOLS  ──(工具执行完)───→ CALL_MODEL
```
