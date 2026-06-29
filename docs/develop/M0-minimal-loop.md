# M0 —— 最小闭环

> **一句话目标**:跑通「5 行示例」+ 全部 M0 测试,用最小代价验证「极薄 + 生产形状」这条路走不走得通。
> **前置依赖**:无(地基阶段)
> **状态**:✅ 已完成(2026-06-27,62 测试全绿;mock_demo 实跑通过)

---

## 一、本阶段要解决的需求

对应 [`DESIGN.md`](../docs/DESIGN.md) 的需求:

| 需求 | 在 M0 的体现 |
|---|---|
| 需求 1 一键写 loop | 内置 agentic loop,实现为**可序列化单步状态机** |
| 需求 2 一键配多厂商(基础) | `LiteLLMProvider` 薄包 + 寻址 `provider/model`(完整打磨留 M1) |
| 需求 4 像 Vue 一样方便人用 | `Agent` + `@tool` + `run()`,守住「5 行」 |
| 需求 5a 并发安全 | **Agent / Session / Chat 三者分离** |
| 需求 5b 防止烧钱 | **熔断四道闸** |
| 需求 9 可测试 | 核心自带 `MockProvider` + 单步可测 |
| 需求 10 轻依赖 | 核心仅 `pydantic + anyio`,litellm 走 extras |

> 需求 5c(恢复)、6(中间件)、7(可观测完整)、8(持久化)、3(Docker)**不在 M0**,但 M0 要为它们**留好形状**(见注意点)。

---

## 二、交付物 / 验收标准

M0 完成 = 下面**全部**为真:

1. ✅ `examples/mock_demo.py` 用 `MockProvider` 跑通一个「模型→调用工具→回填→再回答」的完整多轮 loop,**无需任何 API key、无需联网**。
2. ✅ 「5 行示例」(`examples/minimal.py`)代码成立(有 key 时能真跑;无 key 时能被 mock 测试覆盖)。
3. ✅ `pytest` 全绿,覆盖:IR 序列化往返、Session 可序列化、schema 生成、工具结果序列化器、loop 多轮编排、工具异常自愈、熔断四道闸、Agent/Session 分离的并发安全。
4. ✅ `pip install -e .`(仅核心依赖)即可 import `rein` 并跑 mock_demo。
5. ✅ `RunResult` 能 `model_dump_json()` 序列化(含其内嵌 Session),再 `model_validate_json()` 还原。

---

## 三、开发注意点(坑与约束)⚠️

> 这一节是 M0 最容易写错的地方,动手前务必看。

1. **为可恢复「留形状」,但不实现 resume**:
   - `Session` 必须**完全可序列化**(纯 pydantic,不挂运行期对象/不可序列化字段)。
   - Loop 必须写成 `step(session) -> (session', interrupt?)` 的**单步纯推进**,**所有跨步状态都进 Session**,不许藏在局部变量里。
   - `step()` 的返回签名现在就要带上 `interrupt`(M0 永远返回 `None`),这样 M2 加 resume 时不改签名。
2. **中断点只在「工具执行前」这一个边界**(M0 不会真触发,但 Stage 流转结构要把这个边界留出来)。绝不允许在 token 流中途中断 —— 这是 M1 流式与 M2 恢复正交的前提。
3. **IR 全部 JSON-able**:`ToolCall.arguments` 是 dict;`ToolResult.content` 是 str。工具返回的任意对象,在**写入 Session 前**就要被序列化器文本化(对象不进 Session)。
4. **Provider 延迟实例化 + 可注入**:
   - `Agent(model=...)` 不要在构造时就建 Provider;**首次 run 时**才按 model 字符串懒建 `LiteLLMProvider`。
   - 必须支持 `Agent(provider=MockProvider([...]))` 注入 —— 这样测试无 key 可跑。
   - `litellm` 只在 `LiteLLMProvider` 内部**函数级 import**,绝不在包顶层 import(否则没装 litellm 就 import 不了 rein)。
5. **同步门面 + 异步内核**:
   - 内核全 `async`;`run()` = `asyncio.run(arun())`。
   - `run()` 若检测到当前已在事件循环里(Jupyter/Web),抛明确错误引导用 `arun()`(用 `asyncio.get_running_loop()` 试探)。
6. **同步工具不能阻塞事件循环**:`@tool` 装饰的同步函数,执行时用 `anyio.to_thread.run_sync` 丢线程池;`async def` 工具直接 await。
7. **工具异常绝不抛到 Loop**:`LocalRuntime` 捕获工具内部异常,封装成 `ToolResult(is_error=True, content=错误摘要)` 回填,让模型读到错误自愈。**唯一例外**:权限 `deny` 也走这个短路(返回 is_error 的 ToolResult)。
8. **工具结果序列化器规则**(写入 Session 前):`str`→原样 / `pydantic.BaseModel`→`model_dump_json()` / `dict`·`list`→`json.dumps(ensure_ascii=False)` / 其他→`str()`。
9. **熔断四道闸,M0 全要**:`max_iterations` / `max_tokens`(累计) / `timeout_s`(墙钟) / `detect_loops`。`detect_loops` 的重复计数与上次签名**存在 Session 里**(可序列化)。任一触顶 → 安全终止,`RunResult.stop_reason` 给原因。
10. **权限 M0 只做 `allow` 与 `deny`**;`ask` 抛 `NotImplementedError("ask 权限将在 M2 实现")`,不要假装支持。
11. **schema 生成 M0 范围**:支持 `str/int/float/bool/list/dict` + `pydantic.BaseModel` 参数;无默认值=required;docstring 整体作工具 description。`Literal`/复杂泛型留 M1(M0 遇到未知类型降级为 `"string"`,不报错)。
12. **同轮多工具并发且保序**:`asyncio.gather` 并发执行,结果按原 `tool_calls` 顺序回填。
13. **核心依赖红线**:`src/rein/` 顶层及核心模块只能 import `pydantic` / `anyio` / 标准库。`litellm` 仅在 provider 内部延迟 import。

---

## 四、TodoList

### 4.1 工程脚手架
- [x] `pyproject.toml`:src layout;`name="rein"`;核心依赖 `pydantic>=2`、`anyio`;extras `litellm`/`docker`/`otel`/`dev`(pytest, pytest-asyncio/anyio)
- [x] `pip install -e ".[dev]"` 装好开发环境(venv 已建)

### 4.2 基础层(IR / 配置 / 状态)
- [x] `src/rein/ir.py`:`ToolCall` / `ToolResult` / `Message` / `Usage`(含 `__add__`) / `Completion` / `ToolSpec`
- [x] `src/rein/config.py`:`LoopConfig`(max_iterations / max_tokens / timeout_s / detect_loops / permission)
- [x] `src/rein/session.py`:`Stage` 枚举(CALL_MODEL/RUN_TOOLS/DONE)+ `Session`(messages / usage / stage / pending_tool_calls / iteration / repeat_count / last_signature / done / stop_reason),全部可序列化
- [x] `src/rein/result.py`:`Step` / `RunResult`(`__str__`→output)/ `Interrupt`(占位结构,M2 用)

### 4.3 能力层(工具 / Provider / Runtime)
- [x] `src/rein/tools.py`:`@tool` 装饰器 / `Tool` / `ToolRegistry` / `build_schema()`(注解→JSON Schema)/ `serialize_result()`(结果文本化)
- [x] `src/rein/providers/base.py`:`Provider` Protocol(`complete`,stream 留 M1)
- [x] `src/rein/providers/mock.py`:`MockProvider`(响应列表:`str` 或 `list[ToolCall]`)—— **核心模块**
- [x] `src/rein/providers/litellm.py`:`LiteLLMProvider`(延迟 import litellm,IR↔litellm 互转)
- [x] `src/rein/runtime/base.py`:`Runtime` Protocol
- [x] `src/rein/runtime/local.py`:`LocalRuntime`(执行工具、异常封装、权限 allow/deny、并发保序、同步工具丢线程池)

### 4.4 核心层(熔断 / Loop / 装配)
- [x] `src/rein/circuit.py`:`check_circuit(session, config, start_time) -> stop_reason | None`(四道闸 + 签名计算)
- [x] `src/rein/loop.py`:`step()`(按 stage 推进一步)+ `run()`(驱动循环 + 熔断/取消检查 + 产出 RunResult)
- [x] `src/rein/agent.py`:`Agent`(无状态蓝图,`@tool`,`run/arun`,延迟/可注入 provider)+ `Chat`(会话句柄,持有 Session)
- [x] `src/rein/__init__.py`:导出 `Agent` / `tool` / `LoopConfig` / `RunResult` / `MockProvider` / IR 类型

### 4.5 示例与测试
- [x] `examples/mock_demo.py`:MockProvider 驱动多轮工具调用(无 key 可跑)
- [x] `examples/minimal.py`:5 行示例(需 key)
- [x] `tests/test_ir.py`:IR 序列化往返
- [x] `tests/test_session.py`:Session 可序列化往返 + Stage 流转
- [x] `tests/test_tools.py`:schema 生成 + 序列化器规则 + 同步/异步工具
- [x] `tests/test_circuit.py`:四道闸各自触发 + detect_loops
- [x] `tests/test_loop.py`:MockProvider 多轮编排 / 工具异常自愈 / 终止
- [x] `tests/test_agent.py`:Agent/Session 分离的并发安全(同一 Agent 并发多 Session 不串台)

### 4.6 收尾验证
- [x] `python examples/mock_demo.py` 跑通
- [x] `pytest` 全绿
- [x] `RunResult` 序列化往返通过
- [x] 更新 `README.md` 全局进度 M0 → ✅,本文件状态 → ✅

---

## 五、与设计文档的对应

- 架构与对象模型:`DESIGN.md` §5(IR / 三概念 / 状态机 / RunResult / 实现级细化决策表)
- M0 范围定义:`DESIGN.md` §6
- 关键决策出处:D1(极薄)/ D2(wrap LiteLLM)/ D3(Agent-Session 分离)/ D4(状态机)/ D10(熔断)/ D11–D20(实现细化)
