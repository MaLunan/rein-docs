# Rein 设计文档 —— 需求、决策与理由

> 工作代号 **Rein**(Reins=缰绳,呼应 Harness=马具;名字后续可改)
> 一句话目标:**5 行代码,把任意大模型变成能调工具、自己循环干活的 agent。**
> 内部定位:一个**极薄但生产级**的单-agent harness(智能体运行时)。
> 状态:本文档是对前期全部讨论的**总账**,取代之前反复修改的架构稿。
> 最后更新:2026-06-27

---

## 0. 怎么读这份文档

这不是一份纯技术规格,而是一份**需求驱动的决策记录**。组织方式:

- **§1 缘起** —— 我们最初要做什么。
- **§2 前提决策** —— 讨论一开始就敲定的四个根本选择。
- **§3 需求逐条** —— 每个需求「是什么 / 怎么做 / **为什么这么做**(含否决了哪些替代方案)」。这是全文重心。
- **§4 自我否决精华** —— 我们如何把「大而全」一刀刀砍成「极薄」。
- **§5 收敛后的架构** —— 上述决策的最终结果(架构 + 核心对象 + 关键代码)。
- **§6 M0 边界与路线图** —— 先做什么、不做什么、为什么这么切。
- **§7 开放问题与最大赌注** —— 诚实记录还没定的事。

---

## 1. 项目缘起

### 1.1 最初的诉求(用户原话提炼)

> 做一个像 Vue / Spring Boot 一样的工具框架,**天然支持 Harness 架构**,可以轻松创建一个拥有 Harness 架构思想的程序:
> - **一键写 loop 循环**
> - **一键配置多厂商大模型**
> - **一键衔接 Runtime 运行层**

### 1.2 什么是 Harness 架构

Harness(马具/挽具)= 包裹并驱动大模型的那层运行时。给「野马」(大模型)套上缰绳,让它能拉车干活。它由几块组成:

- **Loop**:心脏。`输入 → 模型补全 → 解析工具调用 → 执行 → 回填 → 再补全 → 终止`。
- **Provider**:把统一格式翻译成各厂商 API,再翻译回来。
- **Runtime**:工具注册、执行、权限、沙箱。
- **Context / Session**:消息历史、状态、记忆。

Claude Code、Cursor、Devin 这类能干活的 agent,底层都是一个朴素但强壮的 harness。

### 1.3 收敛后的一句话定位

经过讨论,从「大而全的 Agent 框架」收窄为:**一个极薄但生产级的单-agent harness**。对外只说那句「5 行把大模型变成能干活的 agent」;「Harness 架构」退居内部设计哲学,不作对外主推广语(因为 90% 开发者不认识这个术语)。

---

## 2. 四个前提决策(讨论开头敲定)

| 决策 | 选择 | 为什么 | 否决了什么 |
|---|---|---|---|
| **语言** | Python | 给 Python 开发者用;AI 生态成熟 | Java(SDK 生态薄)、TS、Go |
| **形态** | 脚手架 CLI + 运行时库 | 既能 `import` 用,也能一键起项目 | 纯库、全栈平台 |
| **差异化路线** | 极薄生产级单-agent harness | 见 §3 需求1 / §4 | 大而全硬刚 LangChain;教学自用;只做现有框架增强层 |
| **部署场景** | 本地 CLI + 服务端 Web **都要** | 决定了「可恢复」必须进核心(见需求 5c) | 只做本地(同步 ask 就够) |

---

## 3. 需求逐条:是什么 / 怎么做 / 为什么

> 格式统一:**【需求】**(用户想要什么) → **【决策】**(怎么做) → **【为什么】**(理由 + 否决的替代方案)。

### 需求 1 —— 一键写 loop

- **【需求】** 不想手写 `while True` + 解析 tool_call + 回填结果这套又脏又易错的循环。
- **【决策】** 内置 agentic loop;但把它实现成**可序列化的单步状态机**:`step(session) -> (session', interrupt?)`,而非一个普通 `for` 循环。
- **【为什么】**
  - 「自己循环调工具」的样板代码每个项目都要重写,且容易写错 —— 这是 harness 的核心价值。
  - 为什么是「状态机」而不是「`for` 循环」?因为部署场景「两者都要」(需求 5c)要求 loop 能暂停/恢复。最直觉的 `async generator` + `yield` 写法**状态在调用栈里、不能序列化,进程一重启就丢**,做不了跨进程恢复。所以基石必须是「所有状态都在可序列化的 Session 里 + 纯单步推进」。
  - 附带红利:**可测试性** —— 可直接构造任意 session 喂给 `step()` 测单步。

### 需求 2 —— 一键配多厂商

- **【需求】** 一行切换底层大模型,不关心各家 API 格式差异。
- **【决策】** Provider 层**直接 wrap LiteLLM**,Rein 只保留一层极薄的「LiteLLM 响应 ↔ Rein IR」转换;寻址对齐 LiteLLM 的 `provider/model`(如 `anthropic/claude-opus-4-8`)。
- **【为什么】**
  - 最初设计是「自研 5 家适配器」。但 LiteLLM 已覆盖 100+ 厂商、统一格式、fallback、成本统计 —— 自研是重复造轮子,且永远追不上它的覆盖面和维护。
  - 「工具调用格式 / 流式分片 / 停止原因」三处归一化的脏活,LiteLLM 已经趟过。
  - 接口仍开放:高级用户可实现自己的 `Provider` 替换默认后端,不锁定 LiteLLM。
  - `litellm` 作为可选依赖(extras),不进核心。

### 需求 3 —— 一键衔接 Runtime 运行层

- **【需求】** 工具的执行环境可插拔,改一行配置就换,业务零改动。
- **【决策】** 只保留 `LocalRuntime`(本地)+ `DockerRuntime`(沙箱);**砍掉 RemoteRuntime**。
- **【为什么】**
  - RemoteRuntime(远程 RPC 执行工具)很重(序列化、网络、安全、协议),而真正需要把工具执行放远程的 agent 极少 —— 典型的投机性 YAGNI。
  - Docker 沙箱保留,因为「跑 LLM 生成的代码/命令」是 coding agent 的真实安全需求。
  - 卖点从「分布式执行」修正为「执行环境可插拔(本地 ↔ 沙箱)」。
  - DockerRuntime 延后到 M4,M0 只做 Local。

### 需求 4 —— 像 Vue 一样方便人用

- **【需求】** 上手极简,5 行能跑通,新手不用先啃懂 harness 原理。
- **【决策】** **渐进式暴露**:`Agent` 无状态蓝图 + `@agent.tool` 装饰器 + `agent.run()`;所有复杂度(状态机/熔断/可恢复)都在底层默认值里。
- **【为什么】**
  - 框架的「薄」指**概念薄、抽象薄、源码能一眼看穿**(反 LangChain 的抽象过载),不是依赖树薄。
  - 必须守住「5 行」:即使加了状态机/熔断/可恢复,`run()` 在 `permission=allow` 时仍一路到底返回结果,新手无感。这是「简单的简单,复杂的可选」(学 FastAPI)。

```python
from rein import Agent

agent = Agent(model="anthropic/claude-opus-4-8")

@agent.tool
def read_file(path: str) -> str:
    "读取文件内容"
    return open(path).read()

print(agent.run("读 README 并总结"))   # 守住 5 行
```

### 需求 5 —— 生产级可用(总纲)

- **【需求】** 不只是 demo,要能真上线(服务端、并发、长任务)。
- **【决策】** 这个「生产级」要求逼出了三个 demo 级框架绝不会碰的硬需求 —— 5a / 5b / 5c。
- **【为什么】** 用「生产级」这把尺子量原设计,发现它骨子里是「demo 级」,缺并发安全、缺成本熔断、缺人工介入,必须补齐。

#### 需求 5a —— 并发安全

- **【需求】** 同一个 agent 在服务端能并发处理多个请求,不串台;多轮对话语义清晰。
- **【决策】** **Agent / Session 分离**:`Agent` 是无状态蓝图(可并发共享、可做全局单例);状态只活在可序列化的 `Session` 里。`agent.run()` 一次性(内部开临时 session),`agent.session()` 返回多轮会话句柄。
- **【为什么】** 原设计里 Agent 同时握「定义」和「运行态 context」,`run()` 就地改 context —— 并发会串台、多轮语义不清。生产级在这点上栽跟头不可接受。

#### 需求 5b —— 防止烧钱失控

- **【需求】** agent 进死循环不能把预算烧光。
- **【决策】** **熔断四道闸**(进 M0):`max_iterations`(次数)/ `max_tokens`(成本)/ `timeout_s`(墙钟)/ `detect_loops`(重复调用检测)。任一触顶安全终止并给出原因。
- **【为什么】** 生产头号事故就是「agent 死循环狂烧 token」。只有 `max_iterations` 远远不够 —— 成本、时间、卡死三种失控它都拦不住。成本低(几个计数器),所以进 M0。

#### 需求 5c —— 人工介入与长任务续跑(HITL)

- **【需求】** 危险操作(退款、删库、跑命令)前要能暂停等人确认;长任务中断后能从断点续跑。
- **【决策】** Loop 在「工具执行前」这个边界产出一个**可序列化的中断态**交还调用方;处理完用 `agent.resume(session, ...)` 恢复。HITL、长任务、断点续跑**本质是同一个机制**。
- **【为什么】**
  - 权限的 `ask`(等人批准)在 CLI 里能 `input()` 阻塞,但在**服务端**意味着 loop 必须能「暂停 → 序列化状态 → 人批准后从断点恢复」。
  - 因为部署场景「两者都要」,所以这套可恢复机制必须进核心(本地场景把同步 ask 当作它的一个特例)。
  - **中断点只设在「一个助手回合完整结束、工具执行之前」**这一个边界 —— 这让流式与恢复正交(见 §4 / 需求 7)。

### 需求 6 —— 生命周期与钩子(用户专门追问的)

- **【需求】** 框架要有生命周期 / 钩子设计(可观测、权限、缓存、护栏都靠它)。
- **【决策】** **收敛为一套中间件**(洋葱模型);「钩子」是它的便捷语法糖,「事件」仅用于只读观测;整套**延后到 M4**。
- **【为什么】**
  - 关键澄清:钩子/中间件的**机制**(调度引擎 + 生命周期模型)是**核心架构**(它是 Loop 的骨架,权限/可观测/压缩都长在上面);而**你写的某个具体钩子**才是扩展。机制在内核,实例是扩展(类比 Spring 的 AOP / Vue 的生命周期)。
  - 最初设计暴露了「中间件 + 钩子 + 事件」三套重叠机制 = 抽象过载(LangChain 被诟病的毛病)。收敛成一套主心智。
  - 为什么延后 M4?它必须在「可恢复 loop」定稿之后设计,才能兼容那条硬约束:**中断只在工具边界、中间件环绕单步且不得跨步持有状态**(否则恢复时调用栈已不在)。在核心闭环前先做扩展点 = 过度设计。

### 需求 7 —— 可观测 / 能调试

- **【需求】** 出问题能查、能回放,知道每一步发生了什么、花了多少 token。
- **【决策】** 核心**不绑任何可观测系统**;`run()` 返回结构化的 `RunResult`(含 `steps` / `usage` / `stop_reason`),这本身就是「可回放的运行记录」。OTel / Langfuse 导出走 extras。
- **【为什么】** 可观测的本质是「留下结构化痕迹」,导出到哪是可选的。一上来就绑 OpenTelemetry 对极薄框架太重,且那是企业级才需要的。结构化结果对象覆盖 90% 调试需求,零外部依赖。

### 需求 8 —— 状态可持久化

- **【需求】** 长任务/会话状态能存下来,断了能恢复。
- **【决策】** **不内置多后端 store**;`Session` 完全可序列化(`model_dump_json()` / `model_validate_json()`),存哪、怎么存(redis/db/文件)是用户的事。最多给一个 `SessionStore` Protocol + 内存/文件参考实现(extras)。
- **【为什么】** Session 已是可序列化的 pydantic 对象,持久化本质就是「dump 成 json 存到任何地方」。框架不该替用户决定基础设施 —— 这才叫薄。

### 需求 9 —— 可测试

- **【需求】** 不烧 token 就能测 loop 逻辑,结果确定。
- **【决策】** 核心自带 `MockProvider`(按预设响应列表返回);加上状态机带来的「单步可测」。
- **【为什么】** agent 逻辑最难测的是「模型→工具→回填」的编排;MockProvider 让它离线、确定性地测。`MockProvider` 在核心(不在 extras),因为它是测试的基础设施。

### 需求 10 —— 安装轻、好上手

- **【需求】** `pip install` 就能开始,不被一堆重依赖劝退。
- **【决策(已演进)】** 最初核心只依赖 `pydantic` + `anyio`,`litellm` 走 extras;**后来为「装完即接入真实大模型」,把 `litellm` 提升为核心依赖**(`docker` / `otel` / `cli` 仍走 extras)。
- **【为什么】** 一度坚持「极薄」把 litellm 放 extras,但「装了却用不了真实模型、还要再 `[litellm]`」太绕;权衡后改为「`pip install rein-agent` 装完即用真实模型」。沙箱/可观测/CLI 仍按需 `pip install "rein-agent[docker]"` 等。

---

## 4. 自我否决精华(怎么从「大而全」砍到「极薄」)

用户要求过「自我推理与自我否决」。下面是几轮红队拷问的结论,这是收敛过程的精华。

**第一批(方向层面):**

1. **存在意义**:原定位「Agent 界的 FastAPI」与 Pydantic AI 几乎一字不差,无差异化 → 收窄到「极薄生产级单-agent harness」,反 LangChain 重抽象。
2. **重复造轮子**:自研多厂商层 = 重复 LiteLLM → 改为 wrap LiteLLM。
3. **伪需求脚手架**:Python 没有 create-vue 文化,重目录树是负价值 → 脚手架只留 `new`(可跑起点)+ `dev`,延后 M5。
4. **抽象过载**:三套扩展机制 → 收敛为一套中间件,延后 M4。
5. **YAGNI**:RemoteRuntime、过早设计扩展点 → 砍掉/延后。
6. **目标用户矛盾**:新手要简单、高手要强大 → 渐进式暴露同时满足。
7. **术语门槛**:「Harness 架构」90% 人不懂 → 对外用大白话。

**第二批(「生产级」逼出来的):**

8. **状态模型**:Agent 持状态会并发串台 → Agent/Session 分离。
9. **熔断**:只有 max_iterations 会烧钱 → 四道闸进 M0。
10. **HITL/可恢复**:同步 ask 在服务端站不住 → loop 做成可序列化状态机。

**第三批(可恢复带来的连锁):**

11. **状态机 vs 生成器**:生成器不能序列化 → 可序列化 Session + 单步推进 才是里子。
12. **流式 × 恢复**:看似冲突,实则正交 —— 因为中断点只在工具边界,不在 token 中途。
13. **持久化**:不造抽象,Session 自序列化。
14. **可观测**:不绑 OTel,给结构化 RunResult。
15. **序列化连锁**:工具可返回任意对象,框架文本化后才进 session(对象不进 session)。
16. **守住 5 行**:加了这么多,验证最小示例仍是 5 行 → 渐进式暴露成立。

**一句话总结**:原设计的病是「还没证明有用,就追求大而全」。砍完之后才像个能服人的框架。

---

## 5. 收敛后的架构(决策的结果)

### 5.1 分层

```
┌─────────────────────────────────────────────┐
│  应用层   你的业务:@tool 定义 / 提示 / run    │
├─────────────────────────────────────────────┤
│  装配层   Agent(无状态蓝图)/ Chat·Session   │
├─────────────────────────────────────────────┤
│  核心层   Loop 单步状态机 / 熔断 / 中断·恢复   │  ← M0 重点
├─────────────────────────────────────────────┤
│  能力层   Provider(wrap LiteLLM)/ Runtime    │
├─────────────────────────────────────────────┤
│  基础层   IR 类型 / 序列化 / RunResult         │
└─────────────────────────────────────────────┘
        扩展层(中间件/事件)= 横切,延后到 M4
```

### 5.2 三个核心概念别混

| 名字 | 是什么 | 可序列化? |
|---|---|---|
| `Agent` | 无状态蓝图:model / system / tools。可并发共享 | —(定义,不可变) |
| `Session` | 可序列化**数据**:messages / usage / stage / 熔断计数 | ✅(pydantic) |
| `Chat` | `agent.session()` 返回的**运行期句柄**:绑定 agent + 持有一个 `Session`,提供 `.run()` | ❌(运行期对象) |

### 5.3 统一内部表示(IR)

**关键约束:IR 必须完全可序列化(pydantic v2)**,这是可恢复状态机的地基。

```python
class ToolCall(BaseModel):
    id: str
    name: str
    arguments: dict[str, Any]                      # JSON-able

class ToolResult(BaseModel):
    tool_call_id: str
    content: str                                   # 文本化后的结果(对象写入前已序列化)
    is_error: bool = False

class Message(BaseModel):
    role: Literal["system", "user", "assistant", "tool"]
    content: str = ""                              # M0 仅文本;多模态延后
    tool_calls: list[ToolCall] | None = None
    tool_call_id: str | None = None

class Usage(BaseModel):
    input_tokens: int = 0
    output_tokens: int = 0
    cost_usd: float | None = None

class Completion(BaseModel):
    message: Message
    usage: Usage
    finish_reason: Literal["stop", "tool_calls", "length", "error"]

class ToolSpec(BaseModel):                          # 喂给模型的工具定义
    name: str
    description: str
    parameters: dict[str, Any]                      # JSON Schema
```

### 5.4 Loop 状态机

```
CALL_MODEL ──(无 tool_calls)──▶ DONE
    │
    │(有 tool_calls,登记 pending_tool_calls)
    ▼
RUN_TOOLS ──(执行完,回填结果,iteration += 1)──▶ CALL_MODEL
```

`step()` 每次只推进一个 stage 转换;`run()` 驱动循环,并在每步边界做熔断检查与取消检查。一轮(iteration)= 一次「模型→工具」。

### 5.5 RunResult

```python
class RunResult(BaseModel):
    status: Literal["done", "interrupted"]
    output: str | None
    session: Session
    steps: list[Step]            # 每步:模型/工具、摘要、耗时、token
    usage: Usage
    stop_reason: str             # done / max_iterations / max_tokens / timeout / loop_detected
    def __str__(self) -> str:    # print(result) 即打印 output
        return self.output or ""
```

### 5.6 实现级细化决策(照着写代码所需的精度)

| 决策 | 内容 |
|---|---|
| 返回值统一 | `agent.run()` / `chat.run()` 返回 `RunResult`,`__str__` 返回 `output` |
| Provider | 延迟实例化(首次 run 懒建 `LiteLLMProvider`)+ 可注入(测试传 `MockProvider`,无需 key) |
| MockProvider | 构造接收响应列表,每项 `str`(文本完成)或 `list[ToolCall]`(调工具),按序消费 |
| 工具结果序列化器 | `str`→原样 / `BaseModel`→`model_dump_json()` / `dict·list`→`json.dumps(ensure_ascii=False)` / 其他→`str()` |
| schema 生成(M0) | 支持 `str/int/float/bool/list/dict` + `BaseModel` 参数;无默认值=required;docstring 作描述;未知类型降级 `string` |
| 并发与不阻塞 | 同轮多工具 `asyncio.gather` 保序并发;同步工具丢 `anyio.to_thread.run_sync` |
| 同步/异步 | `run`(同步门面,内部 `asyncio.run`)+ `arun`(异步);已在事件循环中则报错引导用 `arun` |
| detect_loops | 对每轮工具调用签名连续重复计数,达阈值(默认 3)即停;计数存 Session |
| 权限(M0) | 只实现 `allow`(顺带 `deny` 短路);`ask` 抛 `NotImplementedError` 指向 M2 |
| 核心依赖 | `pydantic` + `anyio`;litellm/docker/otel 走 extras |

---

## 6. M0 边界与路线图

**M0 关键原则:为可恢复「留好形状」,但不在 M0 实现可恢复。** 区分「设计成可序列化」(进 M0,否则地基错)与「实现 resume」(延后,增量不返工)。

### M0 范围(最小闭环,目标:跑通 5 行示例 + 全部 M0 测试)

- ✅ IR + 可序列化 Session(含熔断计数)
- ✅ Loop 单步状态机 `step()`(Stage 流转)
- ✅ `LiteLLMProvider`(薄包,延迟实例化)+ `MockProvider`(核心)
- ✅ `LocalRuntime` + `@tool`(自动 schema + 结果序列化器)
- ✅ `Agent` / `Session` / `Chat` 分离
- ✅ `run()`/`arun()` 同步门面 + 异步内核,返回 `RunResult`
- ✅ 熔断四道闸
- ✅ `permission=allow`(顺带 `deny`)一路到底
- ⏸ **不做**:`resume()` / 持久化 / `ask` 交互 / 流式 / 中间件 / Docker / CLI(形状已留好)

### 路线图

| 阶段 | 内容 |
|---|---|
| **M0 最小闭环** | 上述范围 —— 用事实验证「极薄 + 生产形状」值不值得做 |
| **M1 多厂商打磨** | LiteLLM 寻址/流式/成本统计;fallback;schema 类型补全 |
| **M2 恢复机制** | `resume()` + 中断类型 + `ask` 交互 + SessionStore |
| **M3 上下文 + 可观测** | 滑动窗口/摘要压缩 + RunResult 完善 + OTel adapter(extras) |
| **M4 扩展层** | 中间件(钩子/事件)+ DockerRuntime + 插件系统 |
| **M5 脚手架** | `rein new` / `rein dev` + 模板 + 文档站 |

---

## 7. 开放问题与最大赌注

1. **最大赌注**:Rein 与 Pydantic AI 高度重叠,差异仅「极薄 + 可序列化状态机 + HITL/断点续跑作为一等公民 + 中文生态友好」。这个差异够不够大到让人愿意换,**目前没有证据**。M0 的唯一使命就是用最小闭环验证它。
2. `Interrupt` / `pending_action` 的精确结构(M2)。
3. 流式事件流的完整类型清单(M1)。
4. `SessionStore` Protocol 的最小接口(M2)。
5. 中间件在「单步重建、不跨步持状态」约束下的具体 API 手感(M4)。
6. 框架正式命名(Rein 是工作代号)。
