# 上下文压缩与可观测

长任务会越积越长,迟早撑爆上下文窗口;而生产部署需要把运行痕迹变成可导出的数据。这两件事 Rein 都内置了。

## 上下文压缩

给 Agent 配一个压缩策略,**每次问模型前**自动压缩超窗的历史:

```python
from rein import Agent, SummarizeCompaction, SlidingWindow

# 摘要式:超 token 阈值时,把旧历史折叠成一条摘要,保留近期原文
agent = Agent("anthropic/...", compaction=SummarizeCompaction(max_tokens=8000, keep_recent=6))

# 或滑窗:只保留 system + 最近 N 条
agent = Agent("anthropic/...", compaction=SlidingWindow(max_messages=20))
```

!!! info "压缩是纯变换,不破坏恢复"
    压缩只是对消息列表的纯变换(messages → messages),进出都是普通 `Message`,所以压缩完照样能存盘、照样能 resume。

**摘要器可注入**:`SummarizeCompaction` 默认用机械摘要(不联网、可测,主要降条数);注入一个 Provider 就用 LLM 做真摘要:

```python
summarizer = Agent("anthropic/...")._resolve_provider()  # 或任意 Provider
SummarizeCompaction(max_tokens=8000, summarizer=summarizer)
```

## 估算 token

```python
from rein import estimate_tokens

estimate_tokens(session.messages)   # 近似估算(中文≈1、英文≈0.25/char),用于触发判断
```

---

## 可观测:RunResult 就是运行记录

可观测的核心是**结构化数据,零依赖**。框架本身就产出一份可回放的 `RunResult`:

```python
result = agent.run("...")

result.elapsed_s          # 总墙钟耗时
for s in result.steps:
    print(s.index, s.kind, s.summary, s.duration_s, s.usage, s.is_error)
    # 每步:model 还是 tool、摘要、耗时、token、是否出错
```

## 导出到 OpenTelemetry

把 `RunResult` 导成 OTel trace(一次 run 是父 span,每步是子 span),在 Jaeger / Tempo / Langfuse 里看时间线:

```python
from rein import export_run

result = agent.run("...")
export_run(result)        # 用全局 tracer;或 export_run(result, tracer=my_tracer)
```

!!! note "OTel 走 extras,绝不进核心"
    `pip install "rein-agent[otel]"` 才需要。可观测分两层:「产出结构化数据」是核心(人人都有);「导出到某后端」是 adapter(谁要谁装)。
