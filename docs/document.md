# Rein 项目总览(随时接续对话的入口文档)

> **用法**:每次开新对话,先读本文件;细节再看 `DESIGN.md`(设计)、`code-guide.md`(代码讲解)、`develop/`(开发计划)。
> 最后更新:2026-06-28(M0~M5 框架收官 + 文档站/官网已搭建,143 passed / 5 skipped)

---

## 🎉 0. 当前真实状态:M0~M5 全部完成(框架收官)

**六个里程碑全部落地,跑测试通过**:

- `src/rein/`(23 个模块):ir/config/session/result/tools、providers(base/mock/litellm/fallback)、
  runtime(base/local/docker)、circuit/loop/agent/middleware/compaction/otel/plugins/store/scaffold/cli。
- `tests/`:20 个测试文件。
- ✅ **验证结果**(2026-06-27 实跑):
  - `import rein` 成功(ver 0.0.1,**44 个导出**,新增 create_project/available_templates 等)
  - `pytest` **143 passed / 5 skipped**(skip = 真实厂商冒烟 + OTel + Docker,需 key/extras)
  - `rein --help` / `rein new myagent -t coder` 实跑通过(entry point 已注册)
  - 各阶段关键能力均实测:5 行示例 / 流式 / fallback / HITL resume / 压缩 / 中间件 / 脚手架

**验收全部满足**:M0 五条✅;M1(流式/fallback/真实成本/Literal;DeepSeek 真实冒烟✅);
M2(ask 审批/拒绝自愈/错误即中断/端到端往返);M3(超窗压缩/可序列化可resume/RunResult/OTel);
M4(洋葱中间件/钩子事件/权限即钩子/DockerRuntime/插件/中间件+恢复共存);M5(rein new/dev/模板✅)。

**唯一可选遗留**:Anthropic 真实冒烟待有效 `sk-ant-` key(DeepSeek 已验证框架链路);M5 文档站骨架暂缓。

**常用验证命令(注:本机 `uv` 命令曾不稳定,统一用 `.venv/bin/python`):**
1. `cd /Users/cww/Desktop/myCode/harnessCom`
2. `.venv/bin/python -c "import rein; print(rein.__version__)"`
3. `.venv/bin/python -m pytest framework/tests -q`
4. `.venv/bin/python framework/examples/mock_demo.py`

---

## 1. 关键文件索引

| 文件 | 作用 |
|---|---|
| `docs/document.md`(本文件) | 项目入口:概况 + 真实进度 + 重启指引 |
| `docs/DESIGN.md` | 设计总账:需求 / 决策 / 为什么 / 架构 / IR / 路线图 |
| `docs/code-guide.md` | 代码讲解(小白向):每个模块是什么、干什么、为什么 |
| `develop/README.md` + `develop/M0..M5` | 分阶段开发计划与 TodoList |
| `src/rein/` | 框架源码 |

---

## 2. 这是什么项目

- **一句话**:Rein —— 一个**极薄但生产级**的单-agent harness 框架(Python)。5 行代码把任意大模型变成能调工具、自己循环干活的 agent。
- **缘起**:要做一个像 Vue / Spring Boot 的框架,天然支持 Harness 架构 —— 一键写 loop、一键配多厂商、一键衔接 Runtime。
- **定位收敛**:从"大而全"收窄为极薄单-agent harness(反 LangChain 重抽象;不做多 agent 编排 / RAG 大生态)。
- **最大赌注**:与 Pydantic AI 高度重叠,差异在「极薄 + 可序列化状态机 + HITL/断点续跑 + 中文友好」。M0 验证它。

---

## 3. 核心需求(详见 DESIGN.md §3)

一键写 loop(可序列化单步状态机)/ 一键配多厂商(wrap LiteLLM)/ 一键衔接 Runtime(Local+Docker)/ 像 Vue 一样好用(5 行 + 渐进暴露)/ 并发安全(Agent-Session 分离)/ 防烧钱(熔断四道闸)/ HITL 续跑 / 生命周期钩子(一套中间件,M4)/ 可观测(RunResult)/ 持久化(Session 自序列化)/ 可测试(MockProvider)/ 轻依赖(pydantic+anyio)。

---

## 4. 关键决策速查(详见 DESIGN.md §0)

极薄生产级定位 / wrap LiteLLM / Agent-Session 分离 / 可序列化单步状态机 / 砍 RemoteRuntime 留 Docker / 扩展机制收敛为一套中间件并延后 M4 / 配置以代码为主 toml 可选 / 可观测给 RunResult 不绑 OTel / Session 自序列化不内置 store / 熔断进 M0。

---

## 5. 进度

- [x] 设计定稿 `DESIGN.md`、开发计划 `develop/`、代码讲解 `code-guide.md`
- [x] ✅ **M0 最小闭环(全部完成,62 测试全绿)**
  - [x] 批 1 数据层 / 批 2 工具系统 / 批 3 能力+熔断 / 批 4 串联(loop/agent/导出/examples)
- [x] 🟢 **M1 多厂商打磨(代码完成;DeepSeek 真实冒烟✅,Anthropic 待有效 key)**
  - [x] 批 1 schema 增强 / 批 2 流式全链路 / 批 3 FallbackProvider + 真实成本 + 冒烟
- [x] ✅ **M2 恢复机制(全部完成)**
  - [x] 批 1 ask 审批+resume / 批 2 错误中断+SessionStore / 批 3 run_interactive+need_input
- [x] ✅ **M3 上下文+可观测(全部完成)**
  - [x] 批 1 compaction(滑窗/摘要)+ 自动触发 / 批 2 RunResult 完善 + otel 导出
- [x] ✅ **M4 扩展层(全部完成)**
  - [x] 批 1 中间件引擎 / 批 2 钩子+事件+权限重构 / 批 3 DockerRuntime+插件+兼容回归
- [x] 🎉 **M5 脚手架(全部完成,143 测试全绿,框架收官)**
  - [x] 批 1:`scaffold.py`(create_project + minimal/coder 模板)
  - [x] 批 2:`cli.py`(rein new/dev,typer extras + entry point)+ 文档收尾

---

## 6. M0 实现策略

- **自底向上**:先数据(名词)后行为(动词),每个模块依赖的都已就位,可立即单测。
- **数据与行为分离(核心)**:`Session` 是纯数据(全可序列化),`Loop` 是无状态纯函数 `step(session)->(新session,中断?)`。状态全在 Session、推进是无状态函数 → "暂停=存盘、恢复=喂回 step"。M0 不实现 resume,但按此形状写,M2 零返工。
- **顺序**:批1 数据(ir/config/session/result)→ 批2 工具(tools)→ 批3 能力+熔断(providers/runtime/circuit)→ 批4 串联(loop/agent/导出)+ 示例测试。

---

## 7. 协作约定

- **逐个模块「先讲思路再动手」**:每个模块先讲设计与原因,确认后再写,写完立即验证。
- **一个一个写、每写一个立刻 ls 核实落盘**(本会话教训:Write 回执会假成功,必须 ls 实测;没落盘就重写)。
- **只认事实、不认文本回执**:用 `ls` / 文件大小 / 退出码核实,不信 stdout 文本。
- **代码带详尽中文注释**:每个源码文件给类型 / 函数 / 关键字段加中文注释。
- **每个模块完成后用大白话讲解**:代码 + 测试通过后,用聊天形式讲清「写了啥、为什么」,并追加到 `code-guide.md`。
- **中文交流**;**破坏性 / 不可逆操作先列清楚再等确认**。

---

## 8. 环境与常用命令

- Python 3.14.6(`/opt/homebrew`)、虚拟环境 `.venv`。依赖已装:pydantic 2.13.4 / anyio 4.14.1 / pytest 9.1.1。
- 装依赖:`.venv/bin/python -m pip install -e ".[dev]"`(核心 pydantic+anyio;litellm 走 extras,M0 不装)。
  ⚠️ 本机 `uv` 命令曾不稳定(输出被吞、装不上),统一用 `.venv/bin/python -m ...`。
- 跑测试:`.venv/bin/python -m pytest framework/tests -q`。
- 核实文件:`ls -la 目标目录` 或 `wc -c 文件`(最可靠的真相来源)。

---

## 9. 如何接续对话(给未来的自己)

1. **先读 §0**,跑 `import` 和 `pytest` 复核(预期 143 passed / 5 skipped)。
2. 框架 M0~M5 已收官。后续可做的方向(都非必须):补 Anthropic 真实冒烟(给有效 key)、文档站、更多模板/中间件、发布打包、性能/真实场景打磨。
3. 若继续开发,沿用 §7 协作约定(逐模块先讲后写 + 一个一个写+核实 + 中文注释 + 讲解追加 code-guide)。
4. 设计依据查 `DESIGN.md`,代码讲解查/补 `code-guide.md`。
