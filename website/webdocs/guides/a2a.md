# 暴露为 A2A 服务

把你的 Rein agent 暴露成 [A2A(Agent2Agent)](https://a2a-protocol.org) 协议的 HTTP 服务,让**任何支持 A2A 的 agent**(不限框架/厂商)都能发现并调用它。

实现的能力:**Agent Card 发现** · **Task 状态机** · **Streaming(SSE)** · **Bearer 认证** · **多轮(contextId)**。

## 一行起服务

```python
from rein import Agent, serve_a2a

agent = Agent("anthropic/claude-opus-4-8", system="你是数据分析助手")

serve_a2a(
    agent, name="数据分析助手", port=8000,
    auth_token="my-secret",                 # 可选:Bearer 认证
    # store=FileSessionStore("./sessions"), # 可选:contextId 跨调用多轮记忆
)
```

端点:

| 端点 | 作用 |
|---|---|
| `GET /.well-known/agent.json` | **Agent Card** —— 别的 agent 用它发现你的能力/地址 |
| `POST /` `message/send` | 调用 → 返回 **Task**(状态机) |
| `POST /` `message/stream` | 流式调用 → **SSE** 事件 |
| `POST /` `tasks/get` / `tasks/cancel` | 查询 / 取消 Task |

## Task 状态机

`message/send` 返回一个 **Task**(A2A 的工作单元),状态 `submitted → working → completed`(或 `failed` / `canceled`):

```bash
curl -X POST http://127.0.0.1:8000/ \
  -H "Authorization: Bearer my-secret" -H "Content-Type: application/json" -d '{
  "jsonrpc":"2.0","id":"1","method":"message/send",
  "params":{"message":{"role":"user","parts":[{"kind":"text","text":"分析数据"}]}}}'
```
```json
{"jsonrpc":"2.0","id":"1","result":{
  "kind":"task","id":"<task-id>","contextId":"<ctx-id>",
  "status":{"state":"completed","message":{"role":"agent","parts":[{"kind":"text","text":"……"}]}},
  "artifacts":[{"parts":[{"kind":"text","text":"……"}]}],
  "history":[ ...用户消息 + agent 回复... ]
}}
```

之后可用 `tasks/get`(查询)/ `tasks/cancel`(取消):

```json
{"jsonrpc":"2.0","id":"2","method":"tasks/get","params":{"id":"<task-id>"}}
```

## Streaming(SSE)

`message/stream` 用 Server-Sent Events 流式返回:先 `status-update(working)`,然后逐字 `artifact-update`,最后 `status-update(completed, final)`。基于 `agent.astream`。

```
data: {"jsonrpc":"2.0","id":"s","result":{"kind":"status-update","status":{"state":"working"},"final":false}}

data: {"jsonrpc":"2.0","id":"s","result":{"kind":"artifact-update","artifact":{"parts":[{"kind":"text","text":"你"}]},"append":true}}

data: {"jsonrpc":"2.0","id":"s","result":{"kind":"status-update","status":{"state":"completed"},"final":true}}
```

## 认证

设了 `auth_token`,Agent Card 会声明 `securitySchemes`,所有调用需带 `Authorization: Bearer <token>`,否则返回 `Unauthorized`(JSON-RPC error `-32001`)。

## 多轮(contextId)

传 `store=`,A2A 的 `contextId` 自动对应一个持久化会话,对方多次调用保持上下文(非流式 `message/send`)。存哪由 store 决定,见 [持久化](../reference/persistence.md)。

## 嵌进自己的 Web 框架

`serve_a2a` 是标准库实现的便捷壳。要嵌进 FastAPI / Starlette,直接用 `A2AServer` 的纯方法:

```python
from rein import A2AServer
server = A2AServer(agent, name="...", url="https://my.host", auth_token="...")

# GET  /.well-known/agent.json  → server.agent_card()
# POST / (非 stream)            → server.handle_rpc(body, headers=request.headers)
# POST / message/stream         → async for e in server.stream_events(body): ...(转成 SSE)
```

`agent_card()` / `handle_rpc(dict)->dict` / `stream_events(dict)->async iter` 都不绑 HTTP 框架。

!!! note "范围"
    已实现:发现 / Task 状态机 / streaming(SSE) / 认证 / 多轮。**未内置**(可基于 `A2AServer` 扩展):push notifications、非文本 artifact 类型、task 的 `input-required` 往返、`tasks/resubscribe`。零额外依赖(标准库 `http.server`);生产要更强的 ASGI,把 `A2AServer` 嵌进 FastAPI 即可。

## 反过来:调用别的 A2A agent

把"调远程 A2A agent"做成一个普通工具,你的 agent 就能调用别人的:

```python
import httpx

@agent.tool
def ask_remote_agent(question: str) -> str:
    "向远程 A2A agent 提问"
    resp = httpx.post("https://other-agent.example.com/", json={
        "jsonrpc": "2.0", "id": "1", "method": "message/send",
        "params": {"message": {"role": "user", "parts": [{"kind": "text", "text": question}]}},
    }).json()
    # message/send 返回 Task,从 status.message 取回复
    return resp["result"]["status"]["message"]["parts"][0]["text"]
```

这样就用 [agent 即工具](extensions.md) 的方式接入了整个 A2A 生态。
