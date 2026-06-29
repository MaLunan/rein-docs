# Runtime(工具执行层)

负责「在哪、怎么」执行工具调用 —— 把 `ToolCall` 变成 `ToolResult`。

---

## Runtime(协议)

统一接口(`Protocol`)。

```python
async def execute(self, call, registry, permission="allow") -> ToolResult
async def execute_all(self, calls, registry, permission="allow") -> list[ToolResult]
```

| 方法 | 说明 |
|---|---|
| `execute` | 执行单个工具调用,返回结果(出错也封装成 `ToolResult`,不抛异常) |
| `execute_all` | 并发执行多个,结果按入参顺序返回(**保序**) |

`permission`:`"allow"` 执行 / `"deny"` 拒绝(回 is_error)/ `"ask"` 由 loop 的权限中间件处理(不应直达 runtime)。

---

## LocalRuntime

在本进程执行工具(默认)。

```python
LocalRuntime()
```

行为:

- **找工具**:按名从 registry 取,找不到回 is_error。
- **异常封装**:工具内部异常**不抛到 loop**,封装成 `ToolResult(is_error=True)` 让模型自愈。
- **不阻塞事件循环**:同步函数丢线程池,`async def` 工具直接 await。
- **并发保序**:`execute_all` 用 `asyncio.gather`,结果按原顺序。

```python
import asyncio
from rein import LocalRuntime, ToolCall, Tool, ToolRegistry

reg = ToolRegistry()
reg.add(Tool.from_function(lambda a, b: a + b))  # 假设有个 add
rt = LocalRuntime()
r = asyncio.run(rt.execute(ToolCall(id="1", name="add", arguments={"a": 2, "b": 3}), reg))
r.content   # "5"
```

!!! note
    一般不直接用 —— Agent 默认就用 `LocalRuntime`。

---

## DockerRuntime

在容器沙箱里执行工具(为「跑 LLM 生成的代码」准备)。接口与 LocalRuntime 一致,改配置即切换。

```python
DockerRuntime(
    image: str = "python:3.12-slim",
    *,
    network_disabled: bool = True,
    mem_limit: str = "256m",
    **container_kwargs,
)
```

| 参数 | 说明 |
|---|---|
| `image` | 执行用的镜像 |
| `network_disabled` | 是否切断容器网络(默认 True,保守) |
| `mem_limit` | 内存上限(默认 256m) |
| `**container_kwargs` | 透传给 docker 的其它容器参数 |

!!! warning "务实范围"
    工具是宿主进程的 Python 函数,没法把函数对象塞进容器。DockerRuntime 用 `inspect.getsource` 取函数源码、在容器里 `python -c` 执行 —— **适合纯函数 + 标准库的工具**;依赖闭包 / 第三方库的需要自定义镜像,否则会得到清晰报错。

!!! note "extras + 延迟 import"
    `pip install "rein[docker]"`,且本机要有可用的 docker 守护进程。`docker` 只在真正执行时才 import。

```python
from rein import DockerRuntime
from rein.loop import arun
# 进阶:把 runtime 传给 loop(默认是 LocalRuntime)
result = await arun(session, provider, registry, runtime=DockerRuntime())
```
