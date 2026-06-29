# Provider(模型接入层)

把一段对话发给某个大模型、拿回结果。是 IR 与各厂商之间唯一的翻译边界。教程见 [多厂商与流式](../guides/providers.md)。

---

## Provider(协议)

统一接口(`Protocol`,鸭子类型)。任何对象只要有匹配的 `complete` 就算 Provider,无需继承。

```python
async def complete(self, messages, tools=None, **kwargs) -> Completion
def stream(self, messages, tools=None, **kwargs) -> AsyncIterator[StreamChunk]   # 异步生成器
```

| 方法 | 说明 |
|---|---|
| `complete` | 把对话历史(+ 可选工具)发给模型,返回一次完整 [`Completion`](ir.md#completion) |
| `stream` | 流式版,逐 [`StreamChunk`](ir.md#streamchunk) 产出 |

**写自己的 Provider**:

```python
from rein import Completion, Message, Usage

class MyProvider:
    async def complete(self, messages, tools=None, **kw):
        # ……调你的后端……
        return Completion(
            message=Message(role="assistant", content="答案"),
            finish_reason="stop",
            usage=Usage(),
        )
    async def stream(self, messages, tools=None, **kw):
        yield  # 实现成异步生成器

agent = Agent(provider=MyProvider())
```

---

## MockProvider

不联网、确定性的测试用 Provider(**核心模块**)。按预设「剧本」依次返回。

```python
MockProvider(responses: list, usage_per_call: Usage | None = None)
```

| 参数 | 说明 |
|---|---|
| `responses` | 剧本列表;每项是 `str`(文本回答)或 `list[ToolCall]`(工具调用) |
| `usage_per_call` | 每次调用计入的用量(默认 input=1/output=1,方便测熔断) |

```python
from rein import Agent, MockProvider, ToolCall

agent = Agent(provider=MockProvider([
    [ToolCall(id="1", name="add", arguments={"a": 1, "b": 2})],  # 第1次:调工具
    "答案是 3",                                                  # 第2次:文本
]))
```

剧本用尽后会回一句结束语,**不会让 loop 卡死**。`stream` 把文本逐字发出来模拟分片。

---

## LiteLLMProvider

接入真实大模型(经 LiteLLM,wrap 100+ 厂商)。

```python
LiteLLMProvider(model: str, **default_params)
```

| 参数 | 说明 |
|---|---|
| `model` | LiteLLM 寻址 `"厂商/模型"` |
| `**default_params` | 透传给 litellm 的默认参数(temperature 等) |

```python
from rein import LiteLLMProvider
p = LiteLLMProvider("anthropic/claude-opus-4-8", temperature=0.7)
```

!!! note "延迟 import + extras"
    `litellm` 只在真正调用时才 import(`pip install "rein[litellm]"`)—— 没装也能 `import rein`、用 MockProvider。真实成本会从 litellm 响应取出填进 `Usage.cost_usd`。
    一般不直接构造它:`Agent("厂商/模型")` 会在首次运行时自动懒建。

---

## FallbackProvider

按 `[主, 备...]` 顺序尝试,可重试错误时自动切换。本身也是一个 Provider,对 loop 透明。

```python
FallbackProvider(
    providers: list[Provider],
    *,
    max_retries_per_provider: int = 1,
    base_delay: float = 0.5,
)
```

| 参数 | 说明 |
|---|---|
| `providers` | 非空列表,第 0 个为主,其余为备 |
| `max_retries_per_provider` | 每个 provider 切换前的额外重试次数 |
| `base_delay` | 指数退避基准秒数(`delay = base_delay * 2**attempt`);设 0 关闭等待 |

**只对可重试错误**(限流 429 / 超时 / 5xx)切换;鉴权(401)/ 参数(400)等致命错误直接抛。流式一旦吐过 chunk 就不再切换。

```python
# 一般通过 Agent 用:
agent = Agent("anthropic/...", fallback=["openai/gpt-4o"])

# 也可直接构造:
from rein import FallbackProvider, MockProvider
fp = FallbackProvider([primary, MockProvider(["备用答案"])])
```

`is_retryable(exc) -> bool`(`from rein.providers.fallback import is_retryable`)按 HTTP 状态码 + 异常类名判定;未知错误默认不重试。
