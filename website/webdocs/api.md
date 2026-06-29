# API 速查

`from rein import ...` —— 一站式导出。下面按用途分组。

## Agent

```python
Agent(
    model: str | None = None,         # "厂商/模型",首次 run 时懒建 provider
    *,
    provider=None,                    # 直接注入 Provider(测试用 MockProvider)
    fallback: list | None = None,     # 备用模型(字符串或 Provider),主备自动切换
    tools: list | None = None,        # 工具(裸函数或 Tool)
    system: str | None = None,        # 系统提示
    config: LoopConfig | None = None, # 熔断 + 权限
    compaction=None,                  # 上下文压缩策略
)
```

**运行**

| 方法 | 说明 |
|---|---|
| `agent.run(prompt)` | 同步跑一次 → `RunResult` |
| `await agent.arun(prompt)` | 异步跑一次 |
| `async for c in agent.astream(prompt)` | 流式,逐 `StreamChunk` |
| `agent.resume(session, *, approve=True, answer=None)` | 从中断态恢复 |
| `await agent.aresume(...)` | 异步恢复 |
| `agent.run_interactive(prompt, approver=None)` | CLI 审批循环 |
| `agent.chat()` | 开一个多轮 `Chat` |

**注册扩展**

| 装饰器 | 说明 |
|---|---|
| `@agent.tool` | 注册工具(返回原函数) |
| `@agent.middleware` | 洋葱中间件 `async (ctx, call_next)` |
| `@agent.before_tool` / `@agent.after_tool` | 工具步前/后钩子 |
| `@agent.before_model` / `@agent.after_model` | 模型步前/后钩子 |
| `agent.on(event, handler)` | 只读事件订阅("step" / "tool") |

## 配置

```python
LoopConfig(
    max_iterations=50,       # 轮数闸
    max_tokens=200_000,      # 成本闸(累计 token);None 不限
    timeout_s=120,           # 墙钟闸;None 不限
    detect_loops=True,       # 重复闸开关
    repeat_threshold=3,      # 连续相同调用阈值
    permission="allow",      # allow / ask / deny
)
```

## 结果

```python
RunResult:
  status        # "done" | "interrupted"
  output        # 最终文本 | None
  session       # 完整会话状态(可存盘/resume)
  usage         # Usage(input/output tokens, cost_usd)
  elapsed_s     # 总耗时
  stop_reason   # done / max_iterations / interrupted / ...
  steps         # list[Step](可回放)
  interrupt     # Interrupt | None(中断详情)
```

## 各层组件

| 组 | 导出 |
|---|---|
| **门面** | `Agent` `Chat` `tool` |
| **IR** | `Message` `ToolCall` `ToolResult` `ToolSpec` `Usage` `Completion` `StreamChunk` |
| **状态/结果** | `Session` `Stage` `RunResult` `Step` `Interrupt` `LoopConfig` |
| **工具** | `Tool` `ToolRegistry` |
| **Provider** | `Provider` `MockProvider` `LiteLLMProvider` `FallbackProvider` |
| **Runtime** | `Runtime` `LocalRuntime` `DockerRuntime` |
| **持久化** | `SessionStore` `MemorySessionStore` `FileSessionStore` |
| **压缩** | `CompactionStrategy` `SlidingWindow` `SummarizeCompaction` `estimate_tokens` |
| **可观测** | `export_run` |
| **扩展** | `StepContext` `load_plugins` `plugin_names` |
| **脚手架** | `create_project` `available_templates` |
| **Loop 进阶** | `step` `arun` `run` `astream` `aresume` |

## Extras

```bash
pip install "rein[litellm]"   # 真实厂商模型
pip install "rein[docker]"    # DockerRuntime 沙箱
pip install "rein[otel]"      # OpenTelemetry 导出
pip install "rein[cli]"       # rein new / rein dev
```
