# Agent · Chat · tool

门面层 —— 你日常打交道的那一层。`from rein import Agent, Chat, tool`

---

## `Agent`

无状态的 agent **蓝图**:定义「怎么跑」,每次运行产出独立 `Session`(所以同一个 Agent 可并发用于多个会话而不串台)。

### 构造

```python
Agent(
    model: str | None = None,
    *,
    provider: Provider | None = None,
    fallback: list | None = None,
    tools: list | None = None,
    system: str | None = None,
    config: LoopConfig | None = None,
    compaction: CompactionStrategy | None = None,
)
```

| 参数 | 类型 | 说明 |
|---|---|---|
| `model` | `str` | LiteLLM 寻址 `"厂商/模型"`(如 `"anthropic/claude-opus-4-8"`)。**首次 run 时**才懒建 provider。 |
| `provider` | `Provider` | 直接注入 Provider(测试用 `MockProvider`)。注入后忽略 `model`/`fallback`。 |
| `fallback` | `list` | 备用模型,元素可为模型字符串或 Provider 对象;主模型限流/报错时按序自动切换。 |
| `tools` | `list` | 工具列表,元素可为**裸函数**或 `Tool` 对象,自动登记。 |
| `system` | `str` | 系统提示,放在每次会话最前面。 |
| `config` | `LoopConfig` | 熔断 + 权限配置。默认 `LoopConfig()`。 |
| `compaction` | `CompactionStrategy` | 上下文压缩策略;给定后每次问模型前自动压缩超窗历史。 |

**示例**

```python
from rein import Agent, LoopConfig, MockProvider

# 1) 最简:只给模型
a = Agent("anthropic/claude-opus-4-8")

# 2) 注入 Mock(测试,无 key)
b = Agent(provider=MockProvider(["你好"]))

# 3) 带系统提示 + 主备 + 审批
c = Agent(
    "anthropic/claude-opus-4-8",
    fallback=["openai/gpt-4o"],
    system="你是严谨的助手。",
    config=LoopConfig(permission="ask"),
)
```

!!! note "懒建 + 可注入"
    给 `model` 时,Provider 在**首次运行**才创建(`import litellm` 也延迟到那时)。给 `provider=` 则直接用注入的 —— 这让测试无 key 可跑。两者都没给会在运行时报清晰错误。

---

### `agent.run(prompt)`

```python
def run(self, prompt: str) -> RunResult
```

同步跑一次。新建一份独立 `Session`(system + user),交给 loop 跑完,返回 [`RunResult`](config-result.md#runresult)。

```python
result = agent.run("帮我算 3 加 4")
print(result)            # 直接给答案
print(result.usage)      # 取细节
```

!!! warning "已在事件循环里会报错"
    `run()` = `asyncio.run(arun())`。若当前已在事件循环中(Jupyter / Web 框架),会报错并引导你改用 `await agent.arun(...)` —— 它绝不偷偷嵌套事件循环。

---

### `agent.arun(prompt)`

```python
async def arun(self, prompt: str) -> RunResult
```

`run` 的异步版,异步内核。在已有事件循环的环境(Web/Jupyter)里用它。

```python
result = await agent.arun("你好")
```

---

### `agent.astream(prompt)`

```python
async def astream(self, prompt: str) -> AsyncIterator[StreamChunk]
```

流式跑一次,实时把模型吐字发出来。逐个产出 [`StreamChunk`](ir.md#streamchunk)。

```python
async for chunk in agent.astream("讲个笑话"):
    if chunk.type == "text":
        print(chunk.text, end="", flush=True)
    elif chunk.type == "done":
        print("\n用量:", chunk.usage.output_tokens)
```

!!! info
    流式只关注「实时文本 + 最终用量」;需要完整 `RunResult` / 会话状态请用 `run`/`arun`。只做异步 `astream`(流式本就该异步)。

---

### `agent.resume(session, *, approve=True, answer=None)`

```python
def resume(self, session: Session, *, approve: bool = True, answer: str | None = None) -> RunResult
```

从一个**中断态 session** 恢复执行(同步)。见 [人工审批与断点续跑](../guides/resume.md)。

| 参数 | 说明 |
|---|---|
| `session` | 中断时拿到的 `result.session`(可先存盘再读回) |
| `approve` | `True`=批准执行待办工具;`False`=拒绝(回填错误,模型自愈) |
| `answer` | `need_input` 场景:把补充信息作为 user 消息注入后继续 |

```python
r = agent.run("删库")                       # permission="ask"
if r.status == "interrupted":
    r = agent.resume(r.session, approve=False)
```

---

### `agent.aresume(session, *, approve=True, answer=None)`

`resume` 的异步版。

```python
r = await agent.aresume(r.session, approve=True)
```

---

### `agent.run_interactive(prompt, approver=None)`

```python
def run_interactive(self, prompt: str, approver: Callable | None = None) -> RunResult
```

CLI 友好的「带审批」运行:`permission="ask"` 时每遇审批中断就问一次,自动续跑到 `done`。

| 参数 | 说明 |
|---|---|
| `prompt` | 用户输入 |
| `approver` | 审批回调 `approver(interrupt) -> bool`;默认在终端用 `input()` 问 y/N。测试/自动化可注入。 |

```python
# 终端交互
agent.run_interactive("帮我清理目录")

# 注入自动审批策略(测试)
agent.run_interactive("...", approver=lambda itr: True)
```

!!! note
    它和服务端「产出中断态交给上层 + resume」是**同一套**机制,不是两套。

---

### `agent.chat(session=None)`

```python
def chat(self, session: Session | None = None) -> Chat
```

开一个多轮会话句柄([`Chat`](#chat)),历史跨轮保留。传入 `session` 可从**已有会话**继续 —— 这是企业级无状态服务的关键(见下)。

---

### 注册扩展(装饰器/方法)

这些在对应章节详讲,这里给索引:

| 装饰器/方法 | 作用 | 详见 |
|---|---|---|
| `@agent.tool` | 注册工具(返回原函数) | [写工具](../guides/tools.md) |
| `@agent.middleware` | 洋葱中间件 | [扩展参考](extensions.md) |
| `@agent.before_tool` / `@agent.after_tool` | 工具步前/后钩子 | [扩展参考](extensions.md) |
| `@agent.before_model` / `@agent.after_model` | 模型步前/后钩子 | [扩展参考](extensions.md) |
| `agent.on(event, handler)` | 只读事件订阅 | [扩展参考](extensions.md) |

---

## `Chat`

会话句柄:持有一份持续的 `Session`,支撑多轮对话(历史跨轮保留)。用 `agent.chat()` 创建。

### `chat.send(prompt)` / `chat.asend(prompt)`

```python
def send(self, prompt: str) -> RunResult
async def asend(self, prompt: str) -> RunResult
```

发一轮消息,返回本轮 `RunResult`;会话历史已更新到 `chat.session`。每轮会把单轮运行态(阶段/熔断计数)复位,但**保留历史**。

```python
chat = agent.chat()
chat.send("我叫小明")
r = chat.send("我叫什么?")     # 模型能看到上一轮
print(r.output)               # 你叫小明
```

### `chat.session`

当前的完整 `Session`(随每轮更新),可序列化存盘。

### 企业级:无状态多轮服务

`Chat` 默认是内存句柄(适合脚本/CLI)。在**无状态 Web 服务**里(多用户、跨请求、跨进程),用 `agent.chat(session=...)` 把会话状态从你的 store 进出 —— Agent 无状态蓝图,一个实例并发服务所有用户:

```python
from rein import Agent, MemorySessionStore   # 生产换成你的 RedisSessionStore

agent = Agent("anthropic/...", system="你是客服助手")
store = MemorySessionStore()

def handle(conversation_id: str, user_msg: str) -> str:
    chat = agent.chat(session=store.load(conversation_id))  # None → 自动新建
    result = chat.send(user_msg)
    store.save(conversation_id, chat.session)               # 写回
    return result.output
```

redis 版 store 实现 `save/load` 两个方法即可(见 [持久化参考](persistence.md#sessionstore));完整示例见仓库 `examples/enterprise_chat.py`。

---

## `tool`(模块级)

```python
def tool(fn: Callable) -> Tool
```

把一个普通函数包成 `Tool` 对象,供 `Agent(tools=[...])` 组合使用。

```python
from rein import Agent, tool

@tool
def add(a: int, b: int) -> int:
    "求和"
    return a + b

agent = Agent("...", tools=[add])   # add 此时是 Tool 对象
```

!!! note "和 `@agent.tool` 的区别"
    `@agent.tool` 注册后**返回原函数**(你照常能调用它);模块级 `@tool` 返回的是 `Tool` 对象(用于组合)。想保留函数本体可调用,就用 `@agent.tool`。
