# 多厂商与流式

Rein 的 Provider 层 wrap LiteLLM,覆盖 100+ 厂商。寻址对齐 LiteLLM 的 `厂商/模型`。

## 一行切换厂商

```python
from rein import Agent

Agent("anthropic/claude-opus-4-8")     # Anthropic
Agent("openai/gpt-4o")                 # OpenAI
Agent("deepseek/deepseek-chat")        # DeepSeek
Agent("gemini/gemini-2.0-flash")       # Google
```

业务代码一行不用改。配好对应厂商的环境变量(`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / …)即可。

!!! note "litellm 是可选依赖"
    真实调用需要 `pip install "rein[litellm]"`。核心层只依赖 pydantic + anyio,没装 litellm 也能 `import rein`、用 `MockProvider` 跑测试。

## 测试用 MockProvider(不联网)

```python
from rein import Agent, MockProvider, ToolCall

agent = Agent(provider=MockProvider([
    [ToolCall(id="1", name="add", arguments={"a": 1, "b": 2})],  # 第1次:调工具
    "答案是 3",                                                  # 第2次:文本回答
]))
```

注入 `provider=` 后会忽略 `model`,适合无 key 的测试。

## 流式输出

```python
import asyncio

async def main():
    async for chunk in agent.astream("讲个笑话"):
        if chunk.type == "text":
            print(chunk.text, end="", flush=True)   # 实时吐字
        elif chunk.type == "done":
            print("\n用量:", chunk.usage.output_tokens)

asyncio.run(main())
```

`StreamChunk` 有三种:`text`(增量文本)、`tool_calls`(拼装完整的工具调用)、`done`(结束 + 累计用量)。

!!! info "流式是旁路观测,不改主干"
    流式实时把吐字发给你,但分片收完会拼成完整结果、走和非流式**一模一样**的状态推进。所以流式与「断点恢复」正交。Rein 只做异步 `astream`(流式本就该异步);需要同步拿完整结果用 `run()`。

## 主备自动切换(fallback)

主模型限流 / 超时 / 5xx 时,自动切到备用模型,对你透明:

```python
agent = Agent(
    "anthropic/claude-opus-4-8",
    fallback=["openai/gpt-4o", "deepseek/deepseek-chat"],
)
```

只对**可重试错误**(限流/超时/5xx)切换;鉴权 / 参数错误直接抛(切了也没用)。还带指数退避。

## 真实成本

LiteLLM 算好的美元成本会自动填进 `Usage.cost_usd`:

```python
result = agent.run("...")
print(result.usage.cost_usd)   # 例:0.00123(拿不到则 None)
```

## 用自己的 Provider

接口很简单 —— 任何有 `async def complete(...)` 的对象都算 Provider(鸭子类型),可直接注入,不锁定 LiteLLM。
