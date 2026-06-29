# 工具系统

把普通函数变成模型能调的工具。教程见 [写工具](../guides/tools.md);这里是 API 细节。

---

## Tool

一个工具 = 函数本体 + 元信息(名字 / 说明 / 参数 schema / 是否异步)。运行期对象,不可序列化,活在 Agent 蓝图里。

```python
Tool(fn, name, description, parameters, is_async)
```

### `Tool.from_function(fn, *, name=None, description=None)`

从普通函数构造 Tool:名字默认取函数名,说明默认取 docstring,参数 schema 自动生成,是否异步自动判断。

```python
from rein import Tool

def read_file(path: str) -> str:
    "读取文件"
    return open(path).read()

t = Tool.from_function(read_file)
t.name          # "read_file"
t.description   # "读取文件"
t.is_async      # False
```

### `tool.spec()`

```python
def spec(self) -> ToolSpec
```

生成「给模型看的说明书」(可序列化的 [`ToolSpec`](ir.md#toolspec))。

```python
t.spec().parameters   # {"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"]}
```

---

## ToolRegistry

一组工具的登记册:注册、按名查找、批量生成说明书。

```python
ToolRegistry()
```

| 方法 | 说明 |
|---|---|
| `add(tool)` | 注册一个工具(同名覆盖) |
| `get(name) -> Tool \| None` | 按名取工具,不存在返回 None |
| `specs() -> list[ToolSpec]` | 所有工具的说明书(发给模型) |
| `len(reg)` | 工具数量 |
| `"name" in reg` | 是否包含某工具 |

```python
from rein import Tool, ToolRegistry

reg = ToolRegistry()
reg.add(Tool.from_function(read_file))
len(reg)              # 1
"read_file" in reg    # True
reg.get("read_file")  # Tool
```

!!! note
    一般你不直接碰 `ToolRegistry` —— `@agent.tool` / `Agent(tools=[...])` 已经帮你管好了。需要在多个 Agent 间共享一组工具时才会手动用它。

---

## 自动 schema 生成(规则)

`Tool.from_function` 内部从类型注解生成参数 JSON Schema,支持:

| Python 注解 | JSON Schema |
|---|---|
| `str` / `int` / `float` / `bool` | string / integer / number / boolean |
| `list[T]` | `{"type": "array", "items": ...}` |
| `dict[K, V]` | `{"type": "object", "additionalProperties": ...}` |
| `Literal["a", "b"]` | `{"type": ..., "enum": ["a", "b"]}` |
| `pydantic.BaseModel` | 该模型的 JSON Schema |
| `X \| None` | 取非 None 类型 |
| 无注解 / 不认识 | 降级为 `string`(绝不报错) |

**无默认值的参数 = 必填**(进 `required`);docstring 整体作工具 description。

---

## 结果序列化(规则)

工具返回值在写入会话前被统一转成文本:

| 返回值 | 转成 |
|---|---|
| `str` | 原样 |
| `pydantic.BaseModel` | `model_dump_json()` |
| `dict` / `list` | `json.dumps(ensure_ascii=False)`(中文不转义) |
| 其它 | `str()` |

对象本身不进 Session,只有转出来的文本进 —— 保证 Session 始终可序列化。
