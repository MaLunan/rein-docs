# 中间件 · 钩子 · 沙箱 · 插件

Rein 的扩展性靠**一套机制**:洋葱中间件。钩子是它的糖,事件是它的只读旁路。不搞三套并列的重叠 API。

## 洋葱中间件

在「每一步」前后插入你的逻辑(计时、日志、改输入、拦截):

```python
@agent.middleware
async def timing(ctx, call_next):
    # before:可读改 ctx.session.messages、可短路(不调 call_next)
    import time; t0 = time.monotonic()
    ctx = await call_next(ctx)            # 调下一层,最内层是真正的 step
    # after:可读 ctx.steps / ctx.interrupt
    print(f"{ctx.stage.value} 耗时 {(time.monotonic()-t0)*1000:.1f}ms")
    return ctx
```

- **短路**:不调 `call_next` 直接 return → 跳过这一步。
- **统一 try/except**:把 `call_next` 包在 try 里即可统一兜错。

!!! info "为什么能兼容「中断/恢复」"
    中间件环绕的是**单步 step**(不是整个 loop),每个阶段都重新过一遍栈。所以中间件栈**无状态、每步重建** —— 要持有状态就放进 Session。于是中断→存盘→恢复时,栈能按需重建,不会丢东西。

## 钩子(中间件的糖)

```python
@agent.before_tool      # 工具执行前;返回 False 可短路
async def guard(ctx):
    print("即将执行:", [tc.name for tc in ctx.session.pending_tool_calls])

@agent.after_model      # 问模型后
async def log(ctx):
    print("模型这步:", [s.kind for s in ctx.steps])
```

四种:`@before_model` / `@after_model` / `@before_tool` / `@after_tool`。

## 事件(只读观测)

```python
agent.on("step", lambda ctx: print("走过", ctx.stage.value))
agent.on("tool", lambda ctx: metrics.incr("tool_calls"))
```

事件只用于观测(打日志 / 上报),不改流程 —— 要改流程用中间件/钩子。

## 权限即钩子

Rein 自己的权限(`ask` 审批)就是一个内置中间件实现的 —— loop 里没有任何权限特例。机制统一,你也能用同样的方式做自定义权限策略。

---

## DockerRuntime:容器沙箱

把工具放进容器执行,隔离风险(为「跑 LLM 生成的代码」准备)。接口和 LocalRuntime 一致,改配置即切换:

```python
from rein import Agent, DockerRuntime
from rein.loop import arun
# 进阶:把 runtime 传给 loop(Agent 默认用 LocalRuntime)
```

!!! warning "务实范围"
    工具是宿主进程的 Python 函数,没法把函数对象塞进容器。DockerRuntime 用 `inspect.getsource` 取函数源码在容器里执行 —— **适合纯函数 + 标准库的工具**;依赖闭包/第三方库的需要自定义镜像。`docker` 走 extras(`pip install "rein[docker]"`),默认网络隔离 + 内存上限。

## 插件发现

第三方包在自己的 pyproject 里声明 entry points,运行时自动发现:

```toml
# 第三方包的 pyproject.toml
[project.entry-points."rein.plugins"]
my_provider = "my_pkg:MyProvider"
```

```python
from rein import load_plugins
plugins = load_plugins()   # {"my_provider": <已加载对象>}
```

用标准库 `importlib.metadata`,零额外依赖;坏插件不拖垮整体。
