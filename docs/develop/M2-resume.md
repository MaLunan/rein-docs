# M2 —— 恢复机制(HITL / 断点续跑)

> **一句话目标**:让 loop 能在工具边界**暂停 → 序列化 → 恢复**,从而支持人工审批(HITL)、长任务续跑、错误重试。
> **前置依赖**:M0(必须,依赖其可序列化 Session + 单步状态机的「形状」)
> **状态**:✅ 已完成(2026-06-27,本地单测全绿;HITL 审批 / resume / 错误中断 / SessionStore 均落地)

---

## 一、本阶段要解决的需求

| 需求 | 在 M2 的体现 |
|---|---|
| 需求 5c 人工介入与长任务续跑 | `resume()` + 中断态 + `ask` 交互 |
| 需求 8 状态可持久化 | `SessionStore` Protocol + 内存/文件参考实现 |

---

## 二、交付物 / 验收标准

1. ✅ `result.status == "interrupted"` 时,`result.session` 可序列化存盘,之后 `agent.resume(session, approve=True/False)` 能从断点继续。
2. ✅ `permission="ask"` 在 CLI 下能交互确认;在服务端下表现为产出 `need_approval` 中断态。
3. ✅ 危险工具被 `deny`/未批准时,以 `is_error` ToolResult 回填,loop 继续(模型自愈),不崩。
4. ✅ Provider/工具报错可建模成 `error` 中断态,调用方可选择重试或放弃。
5. ✅ 全程往返(run→中断→存盘→读盘→resume→完成)有端到端测试。

---

## 三、开发注意点(坑与约束)⚠️

1. **中断态必须可序列化**:`Interrupt`(`need_approval` / `need_input` / `error`)及其携带的「恢复所需信息」全部进可序列化结构。
2. **恢复 = 喂回 session**:`resume()` 不是「继续一个挂起的协程」,而是「拿一个 session 数据,从它的 stage 继续 step」。所以**绝不能依赖任何进程内挂起状态**。
3. **错误即中断态**:把 provider/工具的不可自愈错误统一建模成 `error` 中断,让调用方决策(重试/放弃),而不是直接抛异常炸掉整个 run。
4. **同步 ask 是特例**:CLI 里 `input()` 阻塞批准,本质等价于「产出 need_approval → 立即本地应答 → resume」。两条路径共用一套中断/恢复机制,不要写两套。
5. **持久化不绑后端**:只定义 `SessionStore` Protocol(`save(id, session)` / `load(id)`),给内存 + 文件参考实现;redis/db 是用户的事(走 extras 或用户自实现)。
6. **`stop_reason` 增加 `interrupted`**;`RunResult.status` 用 `interrupted`。

---

## 四、TodoList

- [x] `Interrupt` 结构定稿(need_approval / need_input / error,全可序列化;M0 已占形状)
- [x] `step()` 在工具边界产出中断态(ask→need_approval;可重试错误→error)
- [x] `Agent.resume(session, approve=, answer=)` / `aresume` 驱动恢复
- [x] `permission="ask"`:CLI 交互 `run_interactive`(可注入 approver)+ 服务端中断态路径(共用一套)
- [x] 错误→中断态:可重试(限流/超时/5xx)→ error 中断;致命(鉴权/参数)→ 直接抛(经讨论选定)
- [x] `SessionStore` Protocol + `MemorySessionStore` + `FileSessionStore`(标准库,无需 extras)
- [x] 端到端测试:run→中断→存(File/Memory)→读→resume→done
- [x] HITL 测试:审批通过/拒绝/多轮 三条路径 + run_interactive
- [x] 更新 README / document / handoff / code-guide 进度

---

## 五、与设计文档的对应

- `DESIGN.md` §3 需求5c、需求8;§4 自我否决 #10/#11/#13;§6 路线图 M2;§7 开放问题 #2/#4。
