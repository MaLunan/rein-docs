# 统一内部表示(IR)

整个框架内部只流通这一套数据类型;各厂商格式差异都在 Provider 边界翻译成它们。全部是 pydantic 模型(可序列化是地基)。

---

## ToolCall

模型发起的「一次工具调用」请求。

```python
ToolCall:
    id: str
    name: str
    arguments: dict[str, Any]
```

| 字段 | 说明 |
|---|---|
| `id` | 这次调用的唯一标识。结果(`ToolResult`)用同一个 id 指回来。 |
| `name` | 要调用的工具名(对应 `@tool` 的函数名)。 |
| `arguments` | 已解析好的参数 dict(不是 JSON 字符串);内容须 JSON 可序列化。 |

```python
from rein import ToolCall
ToolCall(id="1", name="add", arguments={"a": 1, "b": 2})
```

---

## ToolResult

一次工具调用的执行结果,准备回填给模型。

```python
ToolResult:
    tool_call_id: str
    content: str
    is_error: bool = False
```

| 字段 | 说明 |
|---|---|
| `tool_call_id` | 指回对应的 `ToolCall.id` |
| `content` | 结果文本(永远是字符串;对象会先被序列化) |
| `is_error` | 是否出错。出错时**不抛异常中断 loop**,而是回填错误信息让模型自愈 |

---

## Message

一条对话消息 —— 对话历史就是一个 Message 列表。结构对齐 OpenAI / LiteLLM 通用格式。

```python
Message:
    role: Literal["system", "user", "assistant", "tool"]
    content: str = ""
    tool_calls: list[ToolCall] | None = None
    tool_call_id: str | None = None
```

| role 组合 | 表达 |
|---|---|
| `system` + content | 系统提示 |
| `user` + content | 用户输入 |
| `assistant` + content | 模型纯回答 |
| `assistant` + tool_calls | 模型要调工具 |
| `tool` + tool_call_id + content | 工具结果回填 |

```python
from rein import Message
Message(role="user", content="你好")
Message(role="tool", tool_call_id="1", content="结果")
```

---

## Usage

token 与成本统计。支持 `+` 直接相加(loop 里逐轮累计)。

```python
Usage:
    input_tokens: int = 0
    output_tokens: int = 0
    cost_usd: float | None = None
```

```python
from rein import Usage
total = Usage(input_tokens=10) + Usage(input_tokens=5)   # input_tokens=15
```

---

## Completion

模型「一次返回」的完整结果(对应一轮 CALL_MODEL)。

```python
Completion:
    message: Message
    usage: Usage
    finish_reason: Literal["stop", "tool_calls", "length", "error"]
```

| `finish_reason` | loop 下一步 |
|---|---|
| `stop` | 正常说完 → 结束 |
| `tool_calls` | 要调工具 → 去执行,再继续 |
| `length` | 达到长度上限 |
| `error` | 出错 |

---

## ToolSpec

喂给模型的「工具定义」—— 告诉模型有哪些工具、怎么调。由 `tools.py` 从函数自动生成。

```python
ToolSpec:
    name: str
    description: str
    parameters: dict[str, Any]   # 参数的 JSON Schema
```

---

## StreamChunk

流式输出的「一个增量片段」(M1)。流式是旁路观测,不参与状态推进。

```python
StreamChunk:
    type: Literal["text", "tool_calls", "done"]
    text: str = ""
    tool_calls: list[ToolCall] | None = None
    usage: Usage | None = None
    finish_reason: str | None = None
```

| `type` | 携带 |
|---|---|
| `text` | `text` = 模型新吐的一段文本(增量) |
| `tool_calls` | `tool_calls` = 这一轮拼装完整的工具调用 |
| `done` | `usage`(累计用量)+ `finish_reason`(结束原因) |

```python
async for chunk in agent.astream("..."):
    if chunk.type == "text":
        print(chunk.text, end="")
```

工具调用的逐字分片在 Provider 内部累积拼装,**完整后**才作为 `tool_calls` 暴露 —— 核心层只认完整 `ToolCall`。
