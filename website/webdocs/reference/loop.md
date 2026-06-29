# Loop(进阶:自驱动状态机)

`Agent` 已经把这些包好了。只有当你想**自己驱动**可序列化单步状态机(自定义循环、嵌入到别的运行时、做框架级集成)时才需要直接用。`from rein import step, arun, run, astream, aresume`

---

## step

```python
async def step(session, provider, registry, config, runtime) -> tuple[Session, list[Step], Interrupt | None]
```

推进状态机**一格**(一个 Stage),返回 `(推进后的 session, 本格流水账, 中断或 None)`。

- `CALL_MODEL`:调一次模型 → 累计 usage/iteration → 要调工具则转 RUN_TOOLS,否则结束。
- `RUN_TOOLS`:执行待办工具(并发保序)→ 回填 → 转回 CALL_MODEL。

这是「可测试性」的来源:可直接构造任意 `Session` 喂给它测单步。

---

## arun / run

```python
async def arun(session, provider, registry, config=None, runtime=None,
               compaction=None, middlewares=None) -> RunResult
def run(...同上...) -> RunResult
```

驱动 `step` 循环跑完一次,产出 `RunResult`。`run` 是 `arun` 的同步外壳。

| 参数 | 说明 |
|---|---|
| `session` | 起始会话(可以是新建的,也可以是中断态的) |
| `provider` | 模型接入 |
| `registry` | 工具登记册 |
| `config` | `LoopConfig`(默认 `LoopConfig()`) |
| `runtime` | 工具执行层(默认 `LocalRuntime()`) |
| `compaction` | 上下文压缩策略(可选) |
| `middlewares` | 中间件列表(可选;内置权限中间件总在最内层) |

每步**之前**先查四道熔断闸;问模型前(若有)先压缩;每步经过中间件洋葱。

```python
import asyncio
from rein import arun, run, Session, Message, MockProvider, ToolRegistry

session = Session(messages=[Message(role="user", content="你好")])
result = run(session, MockProvider(["你好呀"]), ToolRegistry())
# 或异步:
result = asyncio.run(arun(session, MockProvider(["你好呀"]), ToolRegistry()))
```

---

## astream

```python
async def astream(session, provider, registry, config=None, runtime=None,
                  compaction=None) -> AsyncIterator[StreamChunk]
```

流式驱动:边跑边把模型吐字实时 yield。不返回 `RunResult`(async 生成器不能带返回值)—— 关注「实时文本 + 最终用量」。

```python
async for chunk in astream(session, provider, registry):
    if chunk.type == "text":
        print(chunk.text, end="")
```

---

## aresume

```python
async def aresume(session, provider, registry, config=None, *,
                  approve=True, answer=None, runtime=None,
                  compaction=None, middlewares=None) -> RunResult
```

从一个**中断态 session** 恢复执行(`agent.aresume` 背后就是它)。先处理掉当前这批待办,再交回 `arun` 继续。

| 参数 | 说明 |
|---|---|
| `approve` | `True`=用 allow 执行待办工具;`False`=回填「被拒绝」,模型自愈 |
| `answer` | `need_input` 场景:把补充信息作为 user 消息注入后继续 |

恢复 = 喂回 session,**不依赖任何进程内挂起状态**。

```python
from rein import aresume
# session 是之前某次 run 返回的中断态 result.session
result = await aresume(session, provider, registry, approve=True)
```
