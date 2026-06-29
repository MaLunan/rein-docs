# M3 —— 上下文与可观测

> **一句话目标**:让长任务不被上下文窗口撑爆,并把「运行痕迹」做成可导出的结构化可观测数据。
> **前置依赖**:M0
> **状态**:✅ 已完成(2026-06-27,本地单测全绿;压缩 + RunResult 完善 + OTel 导出 adapter 均落地)

---

## 一、本阶段要解决的需求

| 需求 | 在 M3 的体现 |
|---|---|
| (上下文管理) | 滑动窗口 / 摘要压缩,超窗自动触发 |
| 需求 7 可观测(完整) | `RunResult` 完善 + OTel / Langfuse 导出(extras) |

---

## 二、交付物 / 验收标准

1. ✅ 上下文超过 token 阈值时自动压缩(默认摘要式:旧轮折叠为摘要,保留近期原文),压缩后 loop 正常继续。
2. ✅ 压缩产物仍可序列化(不破坏 M2 的恢复能力)。
3. ✅ `RunResult.steps` 信息完整(每步模型/工具、耗时、token、错误),可作为「可回放运行记录」。
4. ✅ 可选 `pip install rein[otel]` 后,把运行记录导出到 OpenTelemetry / Langfuse。

---

## 三、开发注意点(坑与约束)⚠️

1. **压缩不破坏可序列化与恢复**:摘要消息也是普通 `Message`,进 Session;压缩是对 `session.messages` 的纯变换,要能在 resume 后依然成立。
2. **压缩触发点**:在 `CALL_MODEL` 之前检查 token 预算;预留 `before_compaction` 概念(具体钩子化留 M4)。
3. **可观测核心零依赖**:核心只产出结构化 `RunResult`;OTel/Langfuse 只是把这份数据**导出**的 adapter,放 extras,绝不进核心。
4. **token 计数**:优先用 Provider/LiteLLM 的计数;无法精确时用近似,但要标明是估算。

---

## 四、TodoList

- [x] `CompactionStrategy` 接口 + `SummarizeCompaction`(摘要,默认机械/可注入 LLM)+ `SlidingWindow`(滑窗)
- [x] 超窗检测与自动触发(`estimate_tokens` 近似;loop CALL_MODEL 前调,arun/astream/aresume 全覆盖)
- [x] 压缩后可序列化 / 可恢复回归测试(含「压缩 + ask resume 共存」)
- [x] `RunResult.elapsed_s` + `Step.duration_s`(工具步)字段补全;Step 已含 token/错误明细
- [x] OTel exporter(`otel.py`,extras,延迟 import):trace=run 父 span,span=每步子 span
- [x] 可观测导出冒烟测试(`test_otel.py`,默认 skip,需 `rein[otel]`)
- [x] 更新 README / document / handoff / code-guide / memory 进度

---

## 五、与设计文档的对应

- `DESIGN.md` §3 需求7;§13 上下文管理;§4 自我否决 #14;§6 路线图 M3。
