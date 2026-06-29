# M1 —— 多厂商打磨

> **一句话目标**:把 M0 的「能调一家模型」升级为「一行切换任意厂商 + 流式 + 成本 + 自动 fallback」。
> **前置依赖**:M0
> **状态**:🟢 基本完成(2026-06-27)。本地单测全绿 84 passed;litellm 已装。
> **真实冒烟**:✅ DeepSeek(OpenAI 兼容)真实调用 + 工具调用多轮**已跑通**,证明全链路正常;
> ⚠️ Anthropic 那条因「无效 key(401)」未过 —— 非代码问题,待提供有效 `sk-ant-...` key 再跑即可。

---

## 一、本阶段要解决的需求

| 需求 | 在 M1 的体现 |
|---|---|
| 需求 2 一键配多厂商(完整) | 寻址、参数透传、能力差异归一化打磨到位 |
| 需求 7 可观测(部分) | 真实成本/token 统计接入 LiteLLM |
| 需求 1 loop(增强) | 流式 `stream` / `astream` |

---

## 二、交付物 / 验收标准

1. ✅ 至少在 Anthropic + OpenAI 兼容(任选一家国产如 DeepSeek)上跑通真实调用冒烟测试。
2. ✅ `agent.stream("...")` 能流式输出文本;流式与状态机正交(中断点仍只在工具边界)。
3. ✅ `fallback=[...]` 在主模型限流/报错时自动切换。
4. ✅ `RunResult.usage` 含真实成本(来自 LiteLLM 的 cost 计算)。
5. ✅ schema 生成支持 `Literal`(→enum)与常见泛型。

---

## 三、开发注意点(坑与约束)⚠️

1. **流式 = 旁路观测,不改状态机主干**:`step` 内部调 provider 流式时,token 通过回调/异步生成器实时发出;但 `step` 的**返回值仍是算完的完整 session**。绝不因流式而把状态推进也改成增量。
2. **流式 tool_call 的拼装**:各家增量 tool_call 分片格式不同,在 Provider 内累积拼完整,完整后才作为一次 `ToolCall` 暴露给 Loop。
3. **fallback 放在 Provider/Router 层**,对 Loop 透明;只对「可重试错误」(限流/5xx/超时)触发,鉴权错误直接抛。
4. **能力差异**:有的模型不支持工具/不支持流式工具 —— 在 Provider 内做能力开关与降级,不要让差异泄漏到核心。
5. **寻址保持对齐 LiteLLM**(`provider/model`),不要自造映射表。

---

## 四、TodoList

- [x] `StreamChunk` IR 类型(text / tool_calls / done;tool_call 分片在 Provider 内拼装,不外暴露 delta)
- [x] `Provider.stream()` / `LiteLLMProvider` 流式实现 + tool_call 分片拼装
- [x] `Agent.astream()` 门面(经讨论:**只做异步 astream**,同步 stream 违背极薄,需同步用 run)
- [x] `loop.astream` 支持流式模式(旁路发 chunk,主干仍单步;有"流式/非流式 session 一致"测试守住)
- [x] fallback router `FallbackProvider`(主→备,可重试错误判定 `is_retryable` + 指数退避)
- [x] 真实成本/token 统计接入(LiteLLM `_hidden_params["response_cost"]` → `Usage.cost_usd`)
- [x] `build_schema` 增强:`Literal`→enum、`dict[K,V]`→additionalProperties、`list[T]`/`Optional`(补测)
- [x] 真实厂商冒烟测试(`test_smoke_providers.py`,`REIN_SMOKE` 开关,默认 skip)
- [x] 流式测试(`test_stream.py`,MockProvider 逐字模拟分片)
- [x] 更新 README / document / handoff 进度

---

## 五、与设计文档的对应

- `DESIGN.md` §3 需求2、需求7;§5.3 IR;§7 Provider;路线图 M1。
