# Rein 代码讲解(小白向)

> 用大白话记录项目里**每个代码模块是什么、干什么、为什么这么写**,配合源码里的中文注释一起看。
> 随开发进度持续追加 —— 每完成一个模块,就把它的讲解补到这里。
> 配套文档:`DESIGN.md`(设计决策)、`document.md`(项目进度入口)、`develop/`(开发计划)。

## 怎么读这个文档

按"依赖关系自底向上"看,前面是"数据"(名词),后面是"行为"(动词):

**ir → config → session → result → tools → providers → runtime → circuit → loop → agent**

---

## 批 1:数据层(项目的"名词")

### `src/rein/ir.py` —— 框架内部的"通用语言"

**作用**:定义一套全框架通用的数据类型。各家大模型(OpenAI / 通义 / 智谱…)的接口格式都不一样,我们在边界把它们统一翻译成这套类型,核心代码就永远只跟这一套"通用语言"打交道,不用操心厂商差异。

**里面有 6 个类型**:
- `ToolCall`:模型说"我要调用某个工具"。
- `ToolResult`:工具执行完的结果。
- `Message`:一条消息(用 `role` 区分系统 / 用户 / 模型 / 工具,谁说的)。
- `Usage`:这次花了多少 token / 多少钱。
- `Completion`:模型一次返回的完整内容。
- `ToolSpec`:给模型看的"工具说明书"(有哪些工具、怎么调)。

**为什么这么写**:全部用 pydantic,所以每个对象都能"存成文字再读回来"(可序列化)—— 这是后面"存档 / 断点续跑"的地基。

---

### `src/rein/config.py` —— Loop 的"控制面板"

**作用**:控制 agent 的循环怎么跑,核心是**防止它失控烧钱**。

**四道保险丝(熔断闸)**:最多转多少轮、最多花多少 token、最多跑多少秒、检测"原地打转"(反复调同一个工具)。任意一道触发就安全停下。

**权限模式**:`allow`(放行)/ `deny`(拒绝)/ `ask`(问人,M2 才做)。M0 默认 `allow`,保证"5 行示例"能一路跑完、不被打断。

**为什么这么写**:生产环境最大的事故就是 agent 死循环把钱烧光,这四道闸是兜底。

---

### `src/rein/session.py` —— agent 的"游戏存档"

**作用**:装一次任务的**全部状态**。

**两样东西**:
- `Stage`(当前在哪个环节):问模型 → 干活 → 收工,三个环节按规则切换。
- `Session`(存档本身):聊天记录、当前环节、待办的工具、转了几轮、花了多少、防打转计数、是否结束。

**最关键的设计**:把所有状态塞进**一个盒子**,这样就能"存档 / 读档"。就像打游戏存档退出、回来读档接着打 —— agent 以后也能干到一半存起来(比如等人批准危险操作),之后接着跑。**M0 还没做存档 / 读档功能本身,但先把状态都收进盒子,以后做就不返工**(这叫"为恢复留形状")。

---

### `src/rein/result.py` —— 干完活的"结果报告"

**作用**:agent 跑完后交给你的报告。

**三样东西**:
- `RunResult`(总报告):`print` 它就直接看到**答案**;要细节再取字段(`steps` 流水账 / `usage` 花费 / `stop_reason` 为什么停 / `session` 完整存档)。一个东西、两种用法。
- `Step`(流水账一行):每做一步(问模型 / 用工具)记一笔,事后能回放整个过程。
- `Interrupt`(中断条):现在用不上,先占位;以后 agent 要等人介入时,用它说明"卡在哪"。

**为什么报告里塞了整个存档**:因为"结果"不只是那句答案,还包括完整过程状态,方便存档、以后接着跑。

---

## 批 2:工具系统(让模型"会用工具")

### `src/rein/tools.py` —— 把普通函数变成"模型能调用的工具"

**作用**:你写一个普通 Python 函数,这个模块就能把它"包装"成模型能理解、能调用的工具——你几乎不用额外做什么。

**里面有四块**:
- `build_schema`(写说明书):看函数的参数和类型注解,自动生成一份"参数说明书"(JSON Schema)给模型看。比如 `def add(a: int, b: int)`,它就生成"要两个整数 a 和 b"。没默认值的参数=必填。
- `serialize_result`(把结果翻译成文字):工具返回的可能是数字、字典、对象……但喂给模型的只能是文字,而且存档也只能存文字。所以统一翻译:字符串原样、pydantic 对象转 JSON、字典/列表转 JSON、其它直接 `str()`。
- `Tool`(一个工具的完整封装):把"函数本身 + 名字 + 说明书 + 是不是异步函数"打包成一个对象。`Tool.from_function(fn)` 一行就能从函数造出来。
- `ToolRegistry`(工具登记册):一个 agent 能用的所有工具放这里,按名字查找。模型说"调 add",就来这里找到 add 这个工具。

**为什么这么写**:核心理念是"让人写工具的成本≈写个普通函数"。注解和 docstring 你本来就会写,框架顺手拿来当说明书,不让你重复劳动。

---

## 批 3:能力层 + 熔断(让 agent"接得上模型、跑得动工具、停得下来")

### `src/rein/providers/` —— 模型接入层(对接各家大模型)

**作用**:负责"把一段对话发给某个大模型,拿回结果"。它是 IR(我们的通用语言)和各家厂商之间唯一的"翻译边界"。

**里面三块**:
- `base.py` 的 `Provider`(接口约定):规定"凡是 Provider,都得有一个 `complete()` 方法"。用的是 Protocol(鸭子类型)——你自己写的类只要"长得像",不用继承就自动算 Provider。
- `mock.py` 的 `MockProvider`(假模型,测试核心):不联网、不烧钱、结果可复现。你给它一个"剧本"(列表),它每次被调用就按顺序吐一项:是文字就当模型回答,是 `[ToolCall]` 就当模型要调工具。剧本演完了还会回一句正常结束,**不让测试卡死**。整个框架的自动化测试全靠它。
- `litellm.py` 的 `LiteLLMProvider`(接真模型):通过 LiteLLM 一个包对接 100+ 厂商。**关键技巧**:`import litellm` 只在真正调用时才执行(函数内部 import),不写在文件顶部——这样没装 litellm 的人照样能 `import rein`、用 MockProvider 跑测试。

**为什么这么写**:把"对接模型"这件最容易厂商绑死的事,收敛到一个薄薄的边界;核心逻辑永远只跟 IR 打交道,换厂商不影响主流程。

---

### `src/rein/runtime/` —— 工具执行层(真正去"干活")

**作用**:模型说"我要调 add(2,3)",由 Runtime 真正去执行,把结果变成 `ToolResult` 回填。

**里面两块**:
- `base.py` 的 `Runtime`(接口约定):规定有 `execute`(执行单个)和 `execute_all`(并发执行多个)两个方法。也是 Protocol。
- `local.py` 的 `LocalRuntime`(在本进程执行,M0 默认):
  - **权限闸**:`deny` 直接拒绝(返回错误结果),`ask` 暂时抛"M2 再做",`allow` 放行。
  - **异常不外抛**:工具内部报错,不让它炸掉整个 loop,而是封装成"出错的结果"(`is_error=True`)回填给模型,让模型读到错误、自己想办法补救(**自愈**)。
  - **不卡事件循环**:同步函数丢到线程池跑,异步函数直接 await。
  - **并发且保序**:同一轮模型要调多个工具时,用 `asyncio.gather` 一起跑,但结果严格按原顺序返回(谁是第 1 个调用,结果就排第 1)。

**为什么这么写**:"在哪执行、怎么执行"被单独抽出来,以后想换成 Docker 沙箱执行(M4),只要再写一个 Runtime,主流程一行不用改。

---

### `src/rein/circuit.py` —— 熔断四道闸(agent 的"安全刹车")

**作用**:agent 最危险的故障是"失控"——死循环狂调工具、烧光预算、卡死不返回。这个模块就是兜底的刹车,loop 每走一步都来问一句"该停了吗?"。

**两个函数**:
- `signature(tool_calls)`(给一组工具调用"按指纹"):把"调了哪个工具、传了什么参数"算成一个字符串,**故意忽略调用 id**(每次 id 都不同)。同名+同参=相同指纹。用它就能判断"模型是不是在原地打转、反复调同一个调用"。
- `check_circuit(session, config, start_time)`(查四道闸):
  1. **轮数闸**:转的轮数到上限了 → `max_iterations`
  2. **成本闸**:累计 token 到上限了 → `max_tokens`(生产头号事故:死循环烧钱,这道最关键)
  3. **超时闸**:从开始跑到现在超时了 → `timeout`(用单调时钟,系统改时间也不误判)
  4. **重复闸**:连续多轮调用完全相同 → `loop_detected`
  - 任一道触顶就返回对应原因(写进 `RunResult.stop_reason`),都没触顶返回 `None`。

**为什么这么写**:`check_circuit` 是个**纯函数**(只看不改),所以好测、能在 loop 任意位置安心调用;重复检测要用的计数都存在 Session 里(可序列化),刹车状态也能随存档一起保存。

---

## 批 4:串联收尾(把零件拼成"能自己跑的 agent")

### `src/rein/loop.py` —— 框架的"心脏"(可序列化单步状态机)

**作用**:把"状态"(Session)和"零件"(模型 / 工具 / 配置)喂进去,一步步推进,直到结束,吐出结果报告。它自己**不存任何状态**——所有状态都在 Session 里进出。

**两个核心函数**:
- `step(...)`(走一格):每次只推进**一个环节**。
  - 在"问模型"环节:调一次模型,累计 token 和轮数;模型要调工具就转去"干活"环节,模型给文本就结束。
  - 在"干活"环节:并发执行所有待办工具,把结果回填成消息,再转回"问模型"。
  - 为什么要一格一格走:这样"工具执行前"这个**边界天然存在**。以后(M2)要做"危险操作先暂停等人批准",就在这一格的开头返回一个"中断条"即可,函数签名都不用改。M0 这里永远返回"无中断"。
- `arun(...)` / `run(...)`(驱动循环):反复调 step 直到结束。**每走一步之前先查四道熔断闸**,一旦触顶就安全停下(绝不多调一次模型、多跑一次工具),最后打包成 `RunResult`。`run` 是 `arun` 的同步外壳;如果发现你已经在事件循环里(Jupyter / Web),会明确报错让你改用 `await arun(...)`,绝不偷偷嵌套。

**为什么这么写**:这是整个框架"反 LangChain"哲学的落点——loop 不是一坨黑盒,而是"无状态函数 + 全状态在 Session"。于是"暂停=把 Session 存盘、恢复=把 Session 喂回 step"这件事天然成立(M0 只搭好形状,M2 才真正实现)。

---

### `src/rein/agent.py` —— 门面(你日常打交道的那一层)

**作用**:守住"5 行代码就能用"的体验,同时保证**并发安全**。

**三样东西**:
- `Agent`(无状态蓝图):只记"这个 agent 该怎么跑"(用哪个模型 / 有哪些工具 / 系统提示 / 熔断配置),**绝不记某一次运行的状态**。每次 `run` 都新建一份独立的 Session。
  - **为什么这是并发安全的关键**:因为 Agent 自己没有可变的"运行态",同一个 Agent 对象可以被同时用于很多个会话,各跑各的 Session,绝不会串台(甲会话读不到乙会话的消息)。这条专门有测试守着。
  - `@agent.tool`:把一个普通函数登记成工具,返回原函数(你照常能调用它)。
  - 模型是"懒建"的:只有真正开跑时才按模型名创建真实 Provider;测试时直接注入 MockProvider,无需 key。
- `Chat`(会话句柄):持有一份**持续的 Session**,支撑多轮对话——历史跨轮保留,模型能看到之前说过的话。每轮开始会把"单轮运行态"(环节 / 是否结束 / 熔断计数)复位,但保留聊天历史。
- `tool`(模块级装饰器):把函数包成 Tool 对象,供 `Agent(tools=[...])` 组合使用。

**为什么这么写**:对应核心设计"Agent / Session / Chat 三者分离"。一句话——**蓝图(Agent)不可变、状态(Session)可序列化、句柄(Chat)管多轮**,三者各司其职,既好用又天然并发安全。

---

> ✅ **至此 M0 最小闭环全部完成**:`examples/mock_demo.py` 无需 key 即可跑通完整多轮 loop;`examples/minimal.py` 是 5 行真实示例;全部测试通过。

---

## M1:多厂商打磨(把"能调一家"升级为"一行切任意厂商 + 流式 + 成本 + 自动 fallback")

### 流式输出(`StreamChunk` + `Provider.stream` + `loop.astream` + `Agent.astream`)

**作用**:让模型像打字机一样**实时**把字吐出来,而不是憋到全部生成完才一次性返回。

**怎么实现的(关键是"旁路"二字)**:
- `StreamChunk`(一个增量片段):`text`(新吐的一段文字)/ `tool_calls`(拼装完整的工具调用)/ `done`(结束,带累计用量)。
- `Provider.stream`:异步生成器,逐段吐 `StreamChunk`。MockProvider 把文本**逐字**发出来模拟分片(测试用);LiteLLMProvider 真流式,并把各家**零碎的工具调用分片**在内部攒完整,完整后才作为一次 `tool_calls` 给出。
- `loop.astream`:**最关键的纪律**——流式只是"旁路观测"。它实时把 `text` 片段透传给你看,但分片收完会**拼成一个完整的 Completion**,然后走和非流式**一模一样**的推进逻辑(`_advance_after_model`)。也就是说,状态机主干一行都没为流式改过。有专门测试比对"流式跑完的 session" 和 "非流式跑完的 session" 完全一致,守住这条纪律。
- `Agent.astream`:门面,`async for c in agent.astream("..."): print(c.text, end="")`。

**为什么只做异步 `astream`、不做同步 `stream`**:流式本质就是异步的;同步流式要靠后台线程+队列桥接,违背"极薄"。需要同步拿完整结果就用 `agent.run()`。

---

### 自动 fallback(`FallbackProvider`)

**作用**:主模型挂了(限流 / 超时 / 服务器 5xx),自动切到备用模型,调用方完全无感。

**怎么实现的**:
- `FallbackProvider` **本身就是个 Provider**,内部持有 `[主, 备1, 备2...]`,按顺序尝试。对 loop 完全透明——loop 根本不知道背后有没有 fallback。
- **只对"可重试错误"切换**:`is_retryable()` 看 HTTP 状态码(429/503/5xx 等可重试;401/403/400 鉴权参数错误直接抛,切了也没用)+ 异常类名兜底;**未知错误默认不重试**(保守,避免盲目重试放大故障)。它**不 import litellm**,纯按状态码+类名判定,核心层零厂商依赖。
- **指数退避**:同一个 provider 可重试几次,每次等待翻倍。
- **流式的红线**:一旦已经吐过字,就**不能再切换**(流出去的收不回),中途失败直接抛。
- 门面:`Agent("anthropic/...", fallback=["openai/gpt-4o", "deepseek/deepseek-chat"])`。

---

### 真实成本 + schema 增强

- **真实成本**:LiteLLM 会把它算好的美元成本塞进响应的 `_hidden_params["response_cost"]`,我们把它取出来填进 `Usage.cost_usd`(取不到就 None,绝不因拿不到成本而让正常调用失败)。
- **`build_schema` 增强**:支持 `Literal["a","b"]`→enum(让模型知道可选值)、`dict[K,V]`→用 additionalProperties 描述值类型;`list[T]`/`Optional` 也补了测试。

---

> ✅ **M1 代码完成**(流式 / fallback / 成本 / schema 增强,本地单测全绿);**真实厂商冒烟测试**(`tests/test_smoke_providers.py`)默认 skip,需配 key + 装 litellm + 设 `REIN_SMOKE=1` 才跑(DeepSeek 已实测跑通)。

---

## M2:恢复机制(让 agent 能"暂停 → 存盘 → 接着跑")

**作用**:危险操作先暂停等人批准(HITL)、长任务跑一半存起来改天接着跑、出错了能重试——这些都靠"在工具边界暂停 + 把状态喂回去恢复"实现。

**为什么 M2 改动很小**:因为 M0 早就把"形状"留好了——`Session` 全可序列化、loop 是单步推进、`step` 的返回值一直带着 `interrupt`(只是之前永远是 None)、中断点早就锁定在"工具执行前"。M2 只是把这些**填上内容**,没有重构主干。这正是"为后续留形状"的回报。

### HITL 人工审批(`permission="ask"` + `resume`)

- 当 agent 配成 `permission="ask"`,`step` 走到"要执行工具"那一格时,**不执行**,而是产出一个 `need_approval` 中断:`agent.run(...)` 返回 `status="interrupted"`,此时 `session` 停在工具执行前、**完全可以存盘**。
- 你看 `result.interrupt.message`(要批准什么)后决定:
  - `agent.resume(session, approve=True)` → 用 allow 执行那批工具,接着跑。
  - `agent.resume(session, approve=False)` → 把"被拒绝"回填给模型,模型**读到拒绝、自己改走安全做法**(自愈)。
- `agent.run_interactive(prompt)`:CLI 里的便捷封装——每遇审批就 `input()` 问 y/N、自动 resume。它和服务端"产出中断态交给上层"用的是**同一套**机制,不是两套。

### 恢复的本质:喂回 session,不靠"挂起的协程"

`resume` **不是**"继续一个卡住的协程",而是"拿一份 session 数据,从它记录的 stage 接着 step"。所以中断态可以**存盘、关机、换台机器、明天再 resume**——只要那份 session 在。这就是"暂停=存盘、恢复=喂回"在 M2 的真正兑现。

### 错误即中断态(可重试的)

模型调用报错时:
- **可重试错误**(限流 429 / 超时 / 5xx)→ 建模成 `error` 中断态,你可以 `resume` 重试(stage 停在"问模型",resume 就重跑这一步)。
- **致命错误**(鉴权 401 / 参数错)→ 直接抛:重试也没用,抛出来最清晰。
- 判定复用了 M1 fallback 的 `is_retryable`,核心层不依赖任何具体厂商。

### 持久化(`SessionStore`)

因为 Session 本身就能序列化,持久化非常朴素:
- `SessionStore` 是个最小接口(`save(id, session)` / `load(id)`)。
- 自带 `MemorySessionStore`(进程内)和 `FileSessionStore`(存成 `{id}.json`),都只用标准库。
- redis / 数据库是你的事——实现这俩方法即可(鸭子类型,无需继承)。"持久化不绑后端"。

**为什么这么写**:M2 把"可中断、可序列化、可恢复"做成了**数据层面的事**(全在 Session + Interrupt 里),而不是"运行时层面的事"(挂起的线程/协程)。于是 HITL、断点续跑、错误重试三件事共用同一套机制,不用各写一套。

---

> ✅ **M2 代码完成**(HITL 审批 / resume 续跑 / 错误中断重试 / SessionStore 持久化,本地单测全绿)。

---

## M3:上下文与可观测(长任务不撑爆 + 运行痕迹可导出)

### 上下文压缩(`compaction.py`)

**作用**:对话越来越长,迟早会超出模型的上下文窗口。压缩就是在"问模型之前"把过长的历史变短,让长任务能一直跑下去。

**关键设计——压缩是"纯变换"**:输入一串消息、输出一串(更短的)消息,进出都是普通 `Message`。所以**压缩完照样能存盘、照样能 resume**,完全不破坏 M2 的恢复能力。

**两种策略**(都实现同一个 `CompactionStrategy` 接口):
- **`SlidingWindow`(滑窗)**:只留 system 提示 + 最近 N 条,更早的直接丢。零成本。还会自动去掉裁剪后"开头的孤儿 tool 消息"(它对应的 assistant 调用被裁掉了,留着会让厂商 API 报错)。
- **`SummarizeCompaction`(摘要)**:超过 token 阈值时,把"最近 K 条之前"的旧历史**折叠成一条摘要消息**,保留近期原文。
  - 默认 `summarizer=None` → **机械摘要**(把旧消息拼接截断,不联网、可测;主要降"条数")。
  - 注入一个 Provider → **LLM 真摘要**(调一次模型概括旧历史;旧历史很长时才显著降 token)。

**怎么用**:`Agent("...", compaction=SummarizeCompaction(max_tokens=8000))`。压缩策略是运行期对象,挂在 Agent 上,**不进 LoopConfig**(那个要保持可序列化)。loop 在每次问模型前自动调一次。

**token 怎么数**:用 `estimate_tokens` 做**近似估算**(中文字符≈1、英文≈0.25/char,偏保守),只用来判断"该不该压",不用于计费。精确计数可后续接 LiteLLM。

### 可观测(`RunResult` 字段 + `otel.py`)

**核心是"结构化数据零依赖"**:框架本身只产出一份结构化的 `RunResult`——
- `elapsed_s`:整次运行的墙钟耗时。
- 每个 `Step`:这步是 model 还是 tool、摘要、耗时(`duration_s`)、token 用量、是否出错。

这份数据本身就是"可回放的运行记录",不依赖任何外部库。

**导出是可选 adapter**(`export_run`,走 extras):把 `RunResult` 翻译成 OpenTelemetry 的 trace——一次 run 是父 span,每个 step 是子 span。这样在 Jaeger / Tempo / Langfuse 里就能看到一次 agent 运行的完整时间线。`opentelemetry` 只在用到时**延迟 import**,没装也不影响核心(`pip install 'rein-agent[otel]'` 才需要)。

**为什么这么写**:可观测分两层——"产出结构化数据"是核心(零依赖、人人都有),"导出到某个后端"是 adapter(谁要谁装)。绝不为了接 OTel 就把 opentelemetry 塞进核心依赖。

---

> ✅ **M3 代码完成**(上下文压缩:滑窗/摘要;可观测:RunResult 完善 + OTel 导出 adapter,本地单测全绿;OTel 冒烟需 `rein-agent[otel]`)。

---

## M4:扩展层(一套机制,撑起所有自定义行为)

**设计哲学**:**机制(中间件调度引擎)= 核心;你写的某个中间件 = 扩展**。M4 只暴露**一种主心智**——洋葱中间件;钩子是它的糖,事件是它的只读旁路。不搞三套并列的重叠 API。

### 洋葱中间件(`middleware.py`)

**作用**:在"每一步"前后插入你自己的逻辑(计时、日志、改输入、拦截…)。

**长这样**(洋葱):
```python
@agent.middleware
async def mw(ctx, call_next):
    # before:可读改 ctx.session.messages、可短路(不调 call_next)
    ctx = await call_next(ctx)   # 调下一层,最内层是真正的 step
    # after:可读 ctx.steps / ctx.interrupt
    return ctx
```

**最关键的设计——兼容"可恢复"**:中间件环绕的是**单步 step**(不是整个 loop),每个阶段都**重新过一遍栈**。所以中间件栈是**无状态的、每步重建**——要持有状态就放进 Session。于是"中断→存盘→恢复"时,中间件栈天然能按需重建,不会丢东西。这正是 M4 必须等 M2(可恢复 loop)定稿后才能做的原因。

无中间件时,loop 等价于"裸 step",所以加了这套机制**不改变默认行为**。

### 钩子 = 中间件的糖

`@before_model`/`@after_model`/`@before_tool`/`@after_tool` 都是中间件的便捷封装(内部按阶段过滤 + 在 call_next 前后调你的函数)。`before_*` 返回 `False` 可短路那一步。

### 事件 = 只读旁路

`agent.on("step", handler)`:每步结束后把上下文推给你**只观测**(打日志/上报),不改流程。也是用一个内置中间件实现的——同一套机制。

### 权限即钩子(重构)

M2 时权限 `ask` 的逻辑写在 `loop.step` 里。M4 把它**搬到一个内置的权限中间件**(`permission_middleware`,总在洋葱最内层、紧贴 step):`ask` 在这里产出中断态并短路。于是 **`loop.step` 里不再有任何权限特例**——架构更统一。`deny` 仍由 Runtime 在执行层处理(那是执行层的职责)。重构后 M2 的所有 resume 测试照样全过,证明行为没变。

### DockerRuntime(沙箱,extras)

**作用**:把工具放进容器里执行,隔离风险(为"跑 LLM 生成的代码"准备)。它实现和 LocalRuntime 一样的接口,`runtime=DockerRuntime()` 即可切换。

**务实范围(诚实说明)**:工具是宿主进程的 Python 函数,没法把函数对象塞进容器。所以它用 `inspect.getsource` 取函数源码、在容器里 `python -c` 执行。**适合纯函数 + 标准库的工具**;依赖闭包/第三方库的需要自定义镜像,否则会得到清晰报错。这是沙箱骨架 + 最常见用例,不是"万能容器执行"。`docker` 走 extras 延迟 import,默认网络隔离 + 内存上限。

### 插件发现(`plugins.py`)

第三方包在自己的 pyproject 里声明 entry points(`rein.plugins` group),`load_plugins()` 就能在运行时自动发现加载——用户不用手动 import。坏插件不拖垮整体(记下错误继续)。

**为什么这么写**:扩展性靠"一套可组合的机制"实现,而不是预先塞满一堆内置功能。机制小而正交(洋葱 + Session),用户能拼出任意行为——这就是"极薄但够用"。

---

> ✅ **M4 代码完成**(中间件/钩子/事件 + 权限重构 + DockerRuntime + 插件发现,本地单测全绿;Docker 冒烟需 `rein-agent[docker]`)。

---

## M5:脚手架(一条命令起项目)

**作用**:让新手用一条命令就得到「能直接跑的起点」,而不是对着空白文件发愁。

### `rein new`(`scaffold.py` + `cli.py`)

```bash
rein new myagent              # 生成 minimal 模板
rein new mybot --template coder   # 生成 coder 模板(带读文件/跑命令工具 + 审批)
```

生成的项目**极简**——就这几个文件,**不堆空目录**(M5 注意点):
```
myagent/
├── main.py          # 能直接跑的起点
├── .env.example     # key 模板
├── README.md
└── rein.toml        # 可选配置示例
```

**关键设计——核心与 CLI 分离**:真正生成文件的是 `scaffold.create_project()`,一个**纯函数**(不依赖 typer),所以好测;`rein new` 只是它的 typer 门面。这样"生成逻辑"和"命令行壳"解耦,核心逻辑能被单测覆盖。

### `rein dev`(热重载)

```bash
rein dev          # 监听 main.py,改了就自动重启
```
用标准库轮询文件 mtime + 重启子进程实现(不引入 watchfiles)。给子进程设 `REIN_DEV=1`,你可在 main.py 里据此挂调试追踪(例:`if os.getenv("REIN_DEV"): agent.on("step", print)`)。

**刻意不做 `rein run`**:跑项目就是 `python main.py`,不搞多余命令(注意点 2)。

**为什么这么写**:脚手架是开发期工具,走 `rein-agent[cli]` extras(typer),**不进核心依赖**;模板给的是"能跑通的最小例子",不是占位骨架——降低上手门槛,又不背离"极薄"。

---

## 🎉 全框架收官:M0 → M5 全部完成

到这里,Rein 的六个里程碑全部落地:

| 阶段 | 交付 | 一句话 |
|---|---|---|
| **M0** | ir/config/session/result/tools/providers/runtime/circuit/loop/agent | 5 行把模型变 agent;可序列化单步状态机 + 熔断四道闸 |
| **M1** | 流式 / fallback / 真实成本 / schema 增强 | 一行切任意厂商(wrap LiteLLM) |
| **M2** | 中断态 / resume / SessionStore | 暂停=存盘、恢复=喂回;HITL 审批 |
| **M3** | compaction / RunResult / OTel | 长任务不撑爆 + 运行痕迹可导出 |
| **M4** | middleware / 钩子 / 事件 / 权限重构 / DockerRuntime / 插件 | 一套洋葱机制撑起所有扩展 |
| **M5** | scaffold / CLI | 一条命令起项目 |

**贯穿始终的两条主线**:① **可序列化的状态(Session)+ 无状态的推进(loop)** —— 让暂停/恢复/压缩/中间件全都成立;② **机制进核心、实例走扩展、重依赖走 extras** —— 守住"极薄但生产级"。
