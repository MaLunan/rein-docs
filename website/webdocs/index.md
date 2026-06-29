---
hide:
  - navigation
  - toc
---

# Rein

<p style="font-size: 1.4rem; font-weight: 300; margin-top: -0.5rem; color: var(--md-default-fg-color--light);">
5 行代码,把任意大模型变成<strong>能调工具、自己循环干活</strong>的 agent。
</p>

<p>
一个<strong>极薄但生产级</strong>的单-agent harness(智能体运行时)。反 LangChain 重抽象,把「单 agent 的 loop + 工具 + 可控 + 可观测」做到极致。
</p>

[快速开始](getting-started.md){ .md-button .md-button--primary }
[看看设计哲学](design.md){ .md-button }

---

## 就 5 行

```python
from rein import Agent

agent = Agent("anthropic/claude-opus-4-8")

@agent.tool
def now() -> str:
    "返回当前日期"
    return "2026-06-28"

print(agent.run("今天几号?用工具查。"))
```

没有 API key?用内置的 `MockProvider` 不联网就能跑通完整的「模型 → 调工具 → 回填 → 再回答」多轮循环。

---

## 为什么用 Rein

<div class="grid cards" markdown>

-   :material-swap-horizontal: __一键多厂商__

    ---

    `Agent("anthropic/...")` 改成 `Agent("openai/gpt-4o")` 就换厂商。底层 wrap LiteLLM,覆盖 100+ 模型;支持 `fallback=[...]` 主备自动切换。

-   :material-content-save-outline: __可暂停 · 可恢复__

    ---

    暂停 = 把会话存盘,恢复 = 把会话喂回去。天然支持人工审批(HITL)、长任务断点续跑、错误重试 —— 跨进程也成立。

-   :material-fuse: __熔断四道闸__

    ---

    轮数 / token / 超时 / 重复检测,任一触顶就安全停下。生产头号事故「agent 死循环烧光预算」,这是兜底。

-   :material-flash: __流式输出__

    ---

    `async for chunk in agent.astream(...)` 实时吐字。流式是旁路观测,不改状态机主干 —— 与恢复机制正交。

-   :material-layers-triple-outline: __洋葱中间件__

    ---

    一套机制撑起所有扩展:中间件、钩子、事件。环绕单步推进、栈每步重建,所以和「中断/恢复」天然兼容。

-   :material-rocket-launch-outline: __一键起项目__

    ---

    `rein new myagent` 生成一个能直接跑的起点(就一个 `main.py`,不堆空目录)。`rein dev` 热重载开发。

</div>

---

## 设计哲学:极薄,但生产级

!!! quote ""
    **机制进核心,实例走扩展,重依赖走 extras。**

- **核心依赖 `pydantic` + `anyio` + `litellm`**(装完即接入真实大模型);docker / opentelemetry / typer 仍是按需安装的可选 extras。
- **可序列化的状态(Session)+ 无状态的推进(loop)**:这一条地基,让暂停、恢复、压缩、中间件全都成立。
- **不做多 agent 编排 / RAG 大生态**:聚焦把「单 agent harness」这件事做到极致。

---

## 它能干什么

<div class="grid" markdown>

:material-robot-outline: **工具型 agent** —— 让模型调你的 Python 函数(查数据库、读文件、调 API),自己循环到完成。
{ .card }

:material-shield-check-outline: **带审批的危险操作** —— `permission="ask"`,执行 `delete_db` 这类工具前暂停等你点头。
{ .card }

:material-clock-outline: **长时间任务** —— 上下文超窗自动压缩;跑一半存盘,改天接着跑。
{ .card }

:material-chart-timeline-variant: **可观测的生产部署** —— 结构化 `RunResult` + OpenTelemetry 导出,一次运行的时间线尽收眼底。
{ .card }

</div>

---

<p style="text-align: center; margin-top: 2rem;">
准备好了吗?
</p>

<p style="text-align: center;">
<a href="getting-started/" class="md-button md-button--primary">5 分钟跑通第一个 agent →</a>
</p>
