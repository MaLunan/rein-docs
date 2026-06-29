# 扩展 · 可观测 · 插件

中间件 / 钩子 / 事件(一套机制)、`StepContext`、OTel 导出、插件发现。教程见 [中间件与扩展](../guides/extensions.md)。

---

## 中间件

```python
@agent.middleware
async def mw(ctx, call_next):
    ...
    ctx = await call_next(ctx)
    ...
    return ctx
```

洋葱式:环绕**单步 step**。`call_next` 是下一层(最内层是真正的 step)。

- **短路**:不调 `call_next` 直接 return → 跳过这一步。
- **统一 try/except**:把 `call_next` 包在 try 里。
- 注册顺序即洋葱顺序(先注册在最外层)。**不得跨步持状态**(要持有就放进 `ctx.session`),以兼容中断/恢复。

---

## StepContext

一次「单步推进」的上下文,在中间件洋葱里层层传递(运行期对象,不序列化)。

```python
StepContext:
    # 输入(可读;session.messages 可改)
    session: Session
    stage: Stage              # 这一步处理的阶段(进入时固定,不随 step 改)
    config: LoopConfig
    provider, registry, runtime
    # 输出(step 后由内核填充,after 阶段可读)
    steps: list[Step]
    interrupt: Interrupt | None
```

```python
@agent.middleware
async def trace(ctx, call_next):
    print("进入", ctx.stage.value)
    ctx = await call_next(ctx)
    print("产出", [s.kind for s in ctx.steps])
    return ctx
```

---

## 钩子(中间件的糖)

```python
@agent.before_model   # CALL_MODEL 前;返回 False 可短路
@agent.after_model    # CALL_MODEL 后
@agent.before_tool    # RUN_TOOLS 前;返回 False 可短路
@agent.after_tool     # RUN_TOOLS 后
async def hook(ctx): ...
```

每个钩子内部转成一个按 `ctx.stage` 过滤的中间件。`before_*` 返回 `False` 短路那一步。

```python
@agent.before_tool
async def guard(ctx):
    names = [tc.name for tc in ctx.session.pending_tool_calls]
    print("即将执行:", names)
```

---

## 事件(只读)

```python
def on(self, event: str, handler: Callable) -> Callable
```

订阅只读事件。`event`:`"step"`(每步后)/ `"tool"`(工具步后)。`handler(ctx)` 只用于观测,不应改流程。

```python
agent.on("step", lambda ctx: print("走过", ctx.stage.value))
agent.on("tool", lambda ctx: metrics.incr("tool_calls"))
```

!!! note "权限即钩子"
    Rein 自己的 `ask` 审批就是一个内置中间件(`permission_middleware`)实现的,总在洋葱最内层 —— loop 里没有任何权限特例。你能用同样的方式做自定义权限策略。

---

## export_run(OTel 导出)

```python
def export_run(result: RunResult, tracer=None) -> None
```

把一次 `RunResult` 导成 OpenTelemetry trace:run 一个父 span,每个 step 一个子 span(带 status / stop_reason / usage / 每步 kind / 耗时 / 是否出错)。

| 参数 | 说明 |
|---|---|
| `result` | 要导出的运行结果 |
| `tracer` | 可选 OTel tracer;不给则用全局 `get_tracer("rein")` |

```python
from rein import export_run
result = agent.run("...")
export_run(result)
```

!!! note "extras"
    `pip install "rein[otel]"` 才需要。`opentelemetry` 延迟 import,没装也不影响核心 —— 可观测核心是结构化 `RunResult`,导出只是 adapter。

---

## 插件发现

```python
def load_plugins(group: str = "rein.plugins") -> dict
def plugin_names(group: str = "rein.plugins") -> list[str]
```

从 entry points 发现第三方插件。`load_plugins` 返回 `{名字: 已加载对象}`(坏插件记下异常,不拖垮整体);`plugin_names` 只列名不加载。

第三方包声明:

```toml
[project.entry-points."rein.plugins"]
my_provider = "my_pkg:MyProvider"
```

```python
from rein import load_plugins
plugins = load_plugins()   # {"my_provider": <已加载对象>}
```

用标准库 `importlib.metadata`,零额外依赖。
