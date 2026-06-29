# 核心概念

理解 Rein 只需要抓住几个东西:**三个对象的分离**、**可序列化的单步状态机**、**熔断四道闸**。

## Agent / Session / Chat 三者分离

这是 Rein 最核心的设计决策(对应并发安全):

| 对象 | 是什么 | 可序列化? |
|---|---|---|
| **`Agent`** | 无状态**蓝图**:定义「怎么跑」(model / 工具 / system / 配置) | — |
| **`Session`** | 可序列化**状态快照**:一次运行的全部状态(消息历史、阶段、用量…) | ✅ |
| **`Chat`** | **会话句柄**:持有一份持续的 Session,支撑多轮对话 | — |

**为什么分离**:`Agent` 不存任何单次运行的状态,所以同一个 Agent 可以被并发用于很多个会话而**互不串台**:

```python
import asyncio
from rein import Agent

agent = Agent("anthropic/claude-opus-4-8")

# 同一个 agent,并发跑两个独立会话,各跑各的 Session,不串台
async def main():
    a, b = await asyncio.gather(agent.arun("问题甲"), agent.arun("问题乙"))
    return a, b
```

多轮对话用 `Chat`(历史跨轮保留):

```python
chat = agent.chat()
chat.send("我叫小明")
chat.send("我叫什么?")   # 模型能看到上一轮
```

---

## 可序列化的单步状态机

Loop 是 Rein 的「心脏」,但它**不是**一个普通的 `while` 循环 —— 而是一个**可序列化的单步状态机**。

```
CALL_MODEL ──(模型给文本)──→ DONE
CALL_MODEL ──(模型要调工具)─→ RUN_TOOLS
RUN_TOOLS  ──(工具执行完)───→ CALL_MODEL
```

关键在于:**所有跨步状态都在可序列化的 `Session` 里,推进是无状态的纯函数** `step(session) -> (新 session, 中断?)`。

这条地基带来一连串能力:

- **暂停 = 把 Session 存盘,恢复 = 把 Session 喂回去** → 人工审批、断点续跑、错误重试。
- **可测试** → 直接构造任意 Session 喂给 `step()` 测单步。
- **流式 / 压缩 / 中间件都正交** → 它们都不破坏这条主干。

!!! info "为什么不用 async generator + yield?"
    那种写法状态在调用栈里、**不能序列化**,进程一重启就丢,做不了跨进程恢复。所以基石必须是「状态全在 Session + 纯单步推进」。

---

## 熔断四道闸

生产里 agent 最危险的故障是「失控」。Rein 内置四道闸,任一触顶就**安全停下**,并在 `RunResult.stop_reason` 说明原因:

```python
from rein import Agent, LoopConfig

agent = Agent(
    "anthropic/claude-opus-4-8",
    config=LoopConfig(
        max_iterations=50,       # ① 轮数闸
        max_tokens=200_000,      # ② 成本闸(累计 token)
        timeout_s=120,           # ③ 墙钟闸
        detect_loops=True,       # ④ 重复闸(连续调用完全相同的工具)
        repeat_threshold=3,
    ),
)
```

第②道(成本闸)针对的就是生产头号事故:agent 死循环烧光预算。

---

## 统一内部表示(IR)

各家大模型的接口格式都不一样。Rein 在 Provider 边界把它们统一翻译成一套 IR,核心层永远只跟这套「通用语言」打交道:

`Message` · `ToolCall` · `ToolResult` · `Usage` · `Completion` · `ToolSpec` · `StreamChunk`

全部是 pydantic 模型 —— 「可序列化」是整个框架的地基。

---

## RunResult:一个对象,两种用法

```python
result = agent.run("...")

print(result)                 # __str__ 直接给最终答案(小白只想看结果)
result.output                 # 最终文本
result.steps                  # 逐步流水账(可回放:每步 model/tool、耗时、token、错误)
result.usage                  # 总 token / 成本
result.stop_reason            # done / max_iterations / interrupted / ...
result.elapsed_s              # 总耗时
result.session                # 完整会话状态(可存盘、可 resume)
```

"结果"不只是那句答案,还包括完整过程状态 —— 方便存档、以后接着跑。

---

## 下一步

- [写工具](guides/tools.md) —— 把普通函数变成模型能调的工具
- [多厂商与流式](guides/providers.md)
- [人工审批与断点续跑](guides/resume.md)
