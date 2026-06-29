# Rein 项目 · 完整会话总结(交接用)

> 这份文档是整段对话的完整交接:项目是什么、用户让我做过的所有事、当前进度、接下来要做的、环境注意、协作约定、关键决策。
> 新对话同步信息时,读这一份 + `document.md` 即可接上。
> 最后更新:2026-06-27

---

## 一、项目是什么

**Rein** —— 一个**极薄但生产级**的单-agent harness(智能体运行时)框架,Python 写。
对外一句话:**5 行代码,把任意大模型变成能调工具、自己循环干活的 agent。**
内部哲学:反 LangChain 重抽象,聚焦把"单 agent 的 loop + 工具 + 可控 + 可观测"做到极致。

---

## 二、用户让我做过的所有事情(按时间顺序)

1. 设计一个像 Vue / Spring Boot 的框架,天然支持 Harness 架构(一键 loop / 一键多厂商 / 一键衔接 Runtime)。
2. 确认要有**生命周期和钩子**设计。
3. 讨论生命周期/钩子**属于"扩展机制"还是"核心架构"**(结论:机制是核心,你写的实例是扩展)。
4. 要求**自我推理 + 自我否决**(框架有什么用、是否好用、设计是否合理、优化点),多轮。
5. 说清"**这个框架有什么用**"。
6. 把设计**改成 v2**。
7. **删除旧文档、重新生成** `DESIGN.md`(需求 + 每个需求点为什么这么做)。
8. 把开发**拆成阶段**,每阶段一个 markdown 放 `develop/`,含需求 / 注意点 / TodoList。
9. **开始写项目**(进入 M0)。
10. 要求每个阶段**用对话方式先讲思路**,让用户把控节奏。
11. 建 `document.md` 作为**项目入口文档**。
12. 诊断"**为什么一打断,文档修改就失败**"。
13. 正式**开始 M0 编码**。
14. 要求**所有代码加详尽中文注释**。
15. **别把"Bash 输出不可靠"写进文档**(那是工具问题,不是项目内容)。
16. 要求**每个模块完成后用大白话讲解**(用户是小白)。
17. 要求把每次讲解**记录到文档**(→ 建了 `code-guide.md`)。
18. 要求**每个模块配正式 pytest**。
19. 要求**一个一个写文件、每写一个检查是否真落盘**,没落盘就重写。
20. **检查代码有没有问题**。
21. 做这份**完整总结**,方便新对话同步。

---

## 三、目前完成的进度

**文档(已落盘):**
- `docs/DESIGN.md` — 设计总账(需求 / 决策 / 为什么 / 架构 / IR / 路线图)
- `docs/document.md` — 项目入口(概况 + 进度 + 重启指引);✅ 已更新为"批 1+2+3 验证通过,45 测试全绿,下一步批 4"
- `docs/code-guide.md` — 代码大白话讲解(批 1+2+3 已全部录入:ir/config/session/result/tools/providers/runtime/circuit;待补 loop/agent)
- `docs/handoff.md` — 本交接文档
- `develop/` — README + M0~M5 七个阶段计划

**代码(M0 全部完成 + M1 代码完成,已写完且已验证通过):**
- M0:`ir`/`config`/`session`/`result`/`tools`/`providers`(base/mock/litellm)/`runtime`(base/local)/`circuit`/`loop`/`agent` + examples
- M1 新增/改动:`providers/fallback.py`(FallbackProvider);`ir.StreamChunk`;`providers.base.stream` 协议;
  Mock/LiteLLM 的 `stream` 实现 + tool_call 分片拼装;`loop.astream`(旁路);`agent.astream` + `Agent(fallback=[...])`;
  `tools.build_schema` 增强(Literal/dict);`litellm` 真实成本(`Usage.cost_usd`)。
- `tests/`:M0 的 10 个 + `test_stream`/`test_fallback`/`test_smoke_providers`(冒烟默认 skip)。
- ✅ **验证结果**(2026-06-27 实跑):`import rein` 成功(ver 0.0.1,**29 个导出**);
  `pytest` **84 passed / 3 skipped**;mock_demo 跑通;`astream` 实测逐字流式;fallback 切换实测通过。
- ✅ **M1 真实冒烟(验收①)**:litellm 已装;DeepSeek 真实调用 + 工具调用多轮**已跑通**(框架全链路验证完成)。
  Anthropic 那条因无效 key(401)未过 —— 非代码问题,提供有效 `sk-ant-...` key 即可补验。

**环境:**
- `.venv` 已重建;依赖已装:pydantic 2.13.4 / anyio 4.14.1 / pytest 9.1.1。

---

## 四、接下来要做的事

- 🎉 **M0 已全部完成;M1 多厂商打磨代码完成**(流式 / fallback / 真实成本 / schema 增强),84 测试全绿;
  `code-guide.md`/`document.md`/`develop/*` 进度均已更新。
- **M1 收尾**:litellm 已装;DeepSeek 真实冒烟(含工具调用)✅ 已跑通,框架全链路验证完成。Anthropic 仅差有效 `sk-ant-...` key(本次用的 key 是 401 无效,非代码问题)。M1 视为基本完成。
- ✅ **M2 恢复机制已全部完成**(2026-06-27,101 测试全绿):
  - HITL:`permission="ask"` 在工具边界产出 `need_approval` 中断;`agent.resume(session, approve=)` 批准/拒绝续跑;`run_interactive` CLI 审批循环。
  - 错误即中断态:可重试(限流/超时/5xx)→ `error` 中断可 resume 重试;致命(鉴权/参数)→ 直接抛(经讨论选定)。
  - 持久化:`SessionStore` Protocol + `MemorySessionStore` + `FileSessionStore`(标准库)。
  - 印证"留好形状"的回报:M2 几乎没改主干,只填了 M0 早留的 `step` interrupt 返回位 + Interrupt 结构。
- ✅ **M3 上下文+可观测已全部完成**(2026-06-27,112 测试全绿):
  - 压缩:`compaction.py`(`SlidingWindow` 滑窗 / `SummarizeCompaction` 摘要,默认机械、可注入 LLM;`estimate_tokens` 近似)。压缩是 messages 纯变换,不破坏可序列化/恢复;loop 在 CALL_MODEL 前自动触发(arun/astream/aresume 全覆盖),挂在 Agent(compaction=) 不进 LoopConfig。
  - 可观测:`RunResult.elapsed_s` + 工具步 `duration_s`;`otel.py` 的 `export_run`(run 父 span + 每步子 span,延迟 import,走 `rein[otel]` extras,冒烟默认 skip)。
- ✅ **M4 扩展层已完成**:洋葱中间件 `middleware.py`(环绕单步 step、栈每步重建 → 兼容恢复)、钩子语法糖、只读事件、**权限重构**(ask 进 `permission_middleware`,loop.step 去特例,M2 测试全过)、`DockerRuntime`(沙箱,docker extras)、`plugins.py`(entry points)。
- 🎉 **M5 脚手架已完成(框架收官)**:`scaffold.create_project`(纯函数)+ `cli.py`(typer,`rein new`/`rein dev`,走 `rein[cli]` extras + `[project.scripts]` entry point)。`rein new myagent -t coder` 实跑通过。模板 minimal/coder 可运行。
- **整框架 M0~M5 全部完成**,143 passed / 5 skipped。
- ✅ **文档站 + 官网已搭建**(2026-06-28,MkDocs Material):
  - `mkdocs.yml`(项目根)+ `webdocs/`(站点源,不碰 docs/ 内部文档):`index.md` 官网首页(landing) + `getting-started` + `concepts` + `guides/`(tools/providers/resume/context/extensions/cli) + `design` + `roadmap` + `api`。
  - 顶层 `README.md`(GitHub 项目介绍门面);`.github/workflows/docs.yml`(push 后自动 gh-deploy 到 GitHub Pages);`pyproject` 加 `docs` extras。
  - 本地预览:`pip install mkdocs-material && mkdocs serve` → http://127.0.0.1:8000。`mkdocs build --strict` 通过、12 页全绿。
- 后续可选方向:Anthropic 真实冒烟(待 key)、发布打包 PyPI、更多模板/中间件、文档站持续完善。

## 九、增量增强(框架收官后)

- ✅ **API 参考教程**(2026-06-28):`webdocs/reference/` 10 页,逐个 API 手写(签名+参数表+示例+注意),挂在文档站「API 参考」组。
- ✅ **Chat 企业级便捷方法**(2026-06-28):`agent.chat(session=...)` 支持传入已有会话(`Chat.__init__` 加 `session` 参数)。
  - 解决的缺口:企业级无状态 Web 服务(多用户/跨请求)不能用内存态 Chat,之前要手动操作 Session + 碰内部 `_resolve_provider`。现在干净三行:`chat = agent.chat(session=store.load(id)); r = chat.send(msg); store.save(id, chat.session)`。
  - 配套:`examples/enterprise_chat.py`(无状态多轮 + Redis store 骨架);`reference/agent.md` 加「企业级无状态多轮」小节。145 passed。
  - RAG 用法(不改框架):检索即工具(`@agent.tool def search_kb`,推荐 agentic)或 `@agent.before_model` 钩子注入;向量库/embedding 用用户自己的(极薄,不内置)。
- ✅ **FAQ 对比页 + 存数据库范例**(2026-06-28):`webdocs/faq.md`(对比 LangChain 多轮会话封装/选型/RAG/选谁);`reference/persistence.md` 加「存数据库」小节(SQLite/Postgres/Redis 三种 SessionStore 范例)。核心:Session `model_dump_json()`/`model_validate_json()` 存哪都行,框架对数据库零依赖零绑定。已实跑 SQLite 多轮+重连续轮验证通过。
- ✅ **A2A 服务端**(2026-06-28):`src/rein/a2a.py` —— `serve_a2a(agent, port=, store=)` 一行把 agent 暴露成 A2A 协议 HTTP 服务,让别的 agent 发现+调用。
  - 两端点:`GET /.well-known/agent.json`(Agent Card 发现)+ `POST /` JSON-RPC `message/send`(调用)。标准库 http.server 零依赖;`A2AServer.agent_card()/handle_rpc()` 纯逻辑可测可嵌 FastAPI。
  - 可选 `store` + A2A `contextId` → 跨调用多轮。导出 `serve_a2a`/`A2AServer`。
  - ✅ **已升级为完整 A2A**(2026-06-28):message/send 返回 **Task 状态机**(submitted→working→completed);`tasks/get`/`tasks/cancel`;`message/stream` **SSE 流式**(基于 astream,逐字 artifact-update);可选 **Bearer 认证**(auth_token,Agent Card 声明 securitySchemes)。`A2AServer.agent_card/handle_rpc/stream_events` 纯方法可测可嵌 FastAPI。
  - 仍未内置(可扩展):push notifications、非文本 artifact、task input-required 往返、resubscribe。
  - 配套:`tests/test_a2a.py`(11 个含 task/auth/stream)、`examples/a2a_server.py`、`webdocs/guides/a2a.md`。**已实跑真实带认证 HTTP 服务**端到端验证(发现/鉴权拒绝/task/tasks_get/cancel/SSE 逐字流式全通过)。**156 passed / 5 skipped**。
  - 多 agent 协作(不改框架):agent 即工具(委派)、流水线、`asyncio.gather`(并行)——已实跑;复杂编排(角色/群聊/路由)用 CrewAI/AutoGen/LangGraph。
- ✅ **多 agent 协作文档**(2026-06-28):`webdocs/guides/multi-agent.md`(委派/流水线/并行 + 跨框架 A2A + 何时用专门框架);`faq.md` 加「能做多 agent/AtoA 协作吗」条目。nav 加「多 agent 协作」。strict build 通过。
- 沿用约定:「逐模块先讲思路再动手 + 一个一个写+核实落盘 + 中文注释 + 讲解追加 code-guide」。

---

## 五、⚠️ 环境注意事项(新对话必看)

- 本会话**工具层时好时坏**:Bash / python / Read 的输出会被吞、改数字、注入乱码;部分 `Write` "假成功"未落盘。
- **`uv` 命令坏了**(输出被吞、装不上包)。替代命令:
  - 装依赖:`.venv/bin/python -m pip install -e ".[dev]"`
  - 跑测试:`.venv/bin/python -m pytest -q`
  - 查文件:`python3 -c "import os; ..."`(用 `os.path`/`open`)
- 曾"假成功"丢失又补回的文件:`pyproject.toml`、`__init__.py`、`.venv`。
- **教训(已写进协作约定)**:只认 `ls` / 退出码 / 文件大小,**不认"写入成功 / 测试通过"的文本回执**;写一个文件就立刻核实落盘。
- **若新对话工具层正常**:第一步重新验证 `cd 项目 && .venv/bin/python -m pytest -q`,确认仍是 28 passed,再继续批 3。

---

## 六、协作约定

- 逐个模块**先讲思路再动手**;**一个一个写、每写一个核实落盘**,没落盘就重写。
- 代码带**详尽中文注释**;每个模块完成后**用大白话讲解并追加到 `code-guide.md`**。
- 只认事实(ls / 退出码 / 文件大小),不认文本回执。
- **全程中文**;破坏性 / 不可逆操作先列清楚再等确认。

---

## 七、关键设计决策(速查,详见 DESIGN.md §0)

- 极薄生产级定位(反 LangChain;不做多 agent 编排 / RAG 大生态)。
- **wrap LiteLLM**(不自研多厂商适配)。
- **Agent(无状态蓝图)/ Session(可序列化状态)/ Chat(会话句柄)三者分离**(否则并发串台)。
- **Loop = 可序列化单步状态机**:状态全在 Session,`step(session)->(新session,中断?)` 无状态推进;为 HITL / 断点续跑留形状(M0 不实现 resume)。
- 熔断四道闸(轮数 / token / 超时 / 重复检测),进 M0。
- 砍 RemoteRuntime,留 Local + Docker(沙箱真需求)。
- 扩展机制收敛为一套中间件,延后到 M4(须在可恢复 loop 定稿后设计)。
- 可观测给结构化 RunResult,OTel 走 extras;持久化靠 Session 自序列化,不内置 store。
- 配置以代码为主,`rein.toml` 可选;核心依赖只 pydantic + anyio,其余 extras。

---

## 八、核心代码结构速查(已实现部分)

- `ir.py`:统一内部类型 `ToolCall / ToolResult / Message / Usage / Completion / ToolSpec`(全 pydantic,可序列化)。
- `config.py`:`LoopConfig`(max_iterations / max_tokens / timeout_s / detect_loops + repeat_threshold / permission)。
- `session.py`:`Stage`(CALL_MODEL/RUN_TOOLS/DONE)+ `Session`(messages/stage/pending_tool_calls/iteration/usage/repeat_count/last_signature/done/stop_reason,纯数据)。
- `result.py`:`Step` / `RunResult`(`__str__` 返回 output)/ `Interrupt`(M2 占位)。
- `tools.py`:`build_schema`(注解→JSON Schema)/ `serialize_result`(结果文本化)/ `Tool`(from_function)/ `ToolRegistry`。
- `__init__.py`:导出上述核心类型(Agent/装饰器等待批 4)。
