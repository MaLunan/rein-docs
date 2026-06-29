# 多 agent 协作

Rein 是**单-agent harness**,刻意**不做内置的多 agent 编排引擎**(角色、群聊、路由、状态图)——这是定位选择。但因为 Rein 的 `Agent` 就是个普通对象、`run()` 返回文本,你能用纯 Python 把多个 agent 拼起来协作。

!!! tip "一个被低估的好处"
    每个子 agent **仍是完整的 Rein agent** —— 各自有自己的熔断四道闸、HITL 审批、断点续跑、可观测,甚至自己的工具和知识库。你是在用一堆「生产级单 agent」拼协作,而不是塞进一个黑盒编排器。

## ① 委派式(agent 即工具)

把一个 agent 包成另一个 agent 的工具,让协调者**自己决定**何时委派:

```python
from rein import Agent

researcher = Agent("anthropic/...", system="你是市场研究员")
writer = Agent("anthropic/...", system="你是商业文案")

boss = Agent("anthropic/...", system="你统筹调研与成文")

@boss.tool
def research(topic: str) -> str:
    "把课题委派给研究员 agent"
    return researcher.run(topic).output       # 一个 agent 调另一个

@boss.tool
def write(brief: str) -> str:
    "把要点交给文案 agent 成文"
    return writer.run(brief).output

boss.run("出一份新能源市场简报")   # boss 自己决定先调研、再成文
```

## ② 流水线(A 的输出 → B 的输入)

固定顺序时,直接串起来:

```python
outline = planner.run("规划文章结构").output
draft   = writer.run(f"按这个结构写:{outline}").output
final   = editor.run(f"润色这篇:{draft}").output
```

## ③ 并行(同时跑,再汇总)

```python
import asyncio

async def main():
    ra, rb = await asyncio.gather(
        optimist.arun("评估这个项目"),
        skeptic.arun("评估这个项目"),
    )
    return ra.output, rb.output
```

## 跨进程 / 跨框架:走 A2A

上面三种是「同进程内」协作。要让你的 agent 被**别的进程 / 别的框架的 agent** 调用,把它[暴露成 A2A 服务](a2a.md):

```python
from rein import serve_a2a
serve_a2a(my_agent, port=8000)        # 别人能发现+调用你
```

反过来,把「调远程 A2A agent」做成一个工具,你的 agent 就能调别人的(还是 agent 即工具):

```python
import httpx

@agent.tool
def ask_remote(question: str) -> str:
    "向远程 A2A agent 提问"
    resp = httpx.post("https://other-agent.example.com/", json={
        "jsonrpc": "2.0", "id": "1", "method": "message/send",
        "params": {"message": {"role": "user", "parts": [{"kind": "text", "text": question}]}},
    }).json()
    return resp["result"]["status"]["message"]["parts"][0]["text"]
```

## 什么时候该用专门的多 agent 框架?

| 需求 | 建议 |
|---|---|
| 层级委派 / 流水线 / 并行 | **Rein 的 agent 即工具**够用,更透明可控 |
| 跨进程/跨框架互操作 | **Rein A2A 服务端**接入 A2A 生态 |
| 复杂 agent 网络(动态路由、群聊、共享黑板、复杂状态机) | 用 **CrewAI / AutoGen / LangGraph**,或在它们里把 Rein agent 当执行节点 |

Rein 不和它们抢「编排」这件事 —— 它把每个**单 agent** 做到生产级,协作交给组合或 A2A。
