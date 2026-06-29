# 写工具

工具就是**让模型能调用的 Python 函数**。Rein 的目标是:写工具的成本 ≈ 写个普通函数。

## 最简单的工具

```python
from rein import Agent

agent = Agent("anthropic/claude-opus-4-8")

@agent.tool
def get_weather(city: str) -> str:
    "查询某个城市的天气"        # docstring → 工具说明,模型据此决定何时调用
    return f"{city}今天晴,25°C"
```

`@agent.tool` 装饰后,`get_weather` **仍是普通函数**(你照常能 `get_weather("北京")` 调用),只是同时被登记成了工具。

## 类型注解 → 参数 schema

Rein 自动从函数的类型注解生成参数说明书(JSON Schema)给模型看,你不用手写:

```python
from typing import Literal

@agent.tool
def search(
    query: str,                       # → string
    limit: int = 10,                  # → integer,有默认值 = 非必填
    order: Literal["asc", "desc"] = "asc",   # → enum,告诉模型可选值
    tags: list[str] | None = None,    # → array;X | None 取非 None 类型
) -> str:
    "搜索"
    ...
```

支持:`str/int/float/bool`、`list[T]`、`dict[K,V]`、`Literal`(→enum)、`pydantic.BaseModel`、`X | None`。没写注解的参数降级为 string,绝不报错。

## 同步 / 异步都行

```python
@agent.tool
async def fetch(url: str) -> str:
    "异步抓取一个 URL"
    async with httpx.AsyncClient() as c:
        return (await c.get(url)).text
```

同步函数会自动丢到线程池执行,不阻塞事件循环;`async def` 工具直接 await。

## 返回值会被自动文本化

工具可以返回任意东西,Rein 在写入会话前统一转成文本:

- `str` → 原样
- `pydantic.BaseModel` → `model_dump_json()`
- `dict` / `list` → JSON(中文不转义)
- 其它 → `str()`

## 工具出错会「自愈」,不会崩

工具内部抛异常**不会炸掉整个 loop** —— 它被封装成一条「出错的结果」回填给模型,让模型读到错误、自己想办法补救:

```python
@agent.tool
def risky() -> str:
    "可能出错的操作"
    raise ValueError("文件不存在")
    # → 模型会收到 "工具执行出错:ValueError: 文件不存在",然后换个做法
```

## 多种注册方式

```python
from rein import Agent, tool

# 1) 装饰器(最常用)
@agent.tool
def a(): ...

# 2) 构造时传入(裸函数或 Tool 对象都行)
def b(x: int) -> int:
    "..."
    return x

agent = Agent("...", tools=[b])

# 3) 模块级 tool() 包成 Tool 对象,供组合
mytool = tool(b)
```

## 危险工具要审批?

给工具配上 `permission="ask"`,执行前会暂停等人批准 —— 见 [人工审批与断点续跑](resume.md)。

```python
from rein import Agent, LoopConfig

agent = Agent("...", config=LoopConfig(permission="ask"))

@agent.tool
def delete_file(path: str) -> str:
    "删除文件(危险)"
    ...
```
