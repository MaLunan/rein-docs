# 路线图

六个里程碑全部完成。每个阶段都「先做后验」,跑通了才进下一个。

!!! success "M0 → M5 全部完成"
    143 passed / 5 skipped(skip = 真实厂商冒烟 + OTel + Docker,需 key/extras)。

| 阶段 | 名称 | 交付 | 状态 |
|---|---|---|---|
| **M0** | 最小闭环 | ir/config/session/result/tools/providers/runtime/circuit/loop/agent;5 行示例 + mock 跑通 | ✅ |
| **M1** | 多厂商打磨 | 流式 / fallback / 真实成本 / schema 增强 | ✅ |
| **M2** | 恢复机制 | 中断态 / resume / HITL / SessionStore 持久化 | ✅ |
| **M3** | 上下文+可观测 | 压缩(滑窗/摘要)/ RunResult 完善 / OTel 导出 | ✅ |
| **M4** | 扩展层 | 中间件/钩子/事件 / 权限重构 / DockerRuntime / 插件 | ✅ |
| **M5** | 脚手架 | `rein new` / `rein dev` + 模板 | ✅ |

## 每个阶段在干什么

=== "M0 最小闭环"

    用最小代价验证「极薄 + 生产形状」走不走得通。落点:可序列化单步状态机 + 熔断四道闸 + Agent/Session/Chat 分离。**5 行示例成立**。

=== "M1 多厂商"

    把「能调一家」升级为「一行切任意厂商」。wrap LiteLLM;流式只做异步 astream(旁路不改主干);fallback 主备切换;真实成本进 Usage。

=== "M2 恢复"

    兑现 M0 埋的伏笔:`step` 早就带了 `interrupt`、中断点早就锁在工具边界。M2 把它们填上 —— HITL 审批、断点续跑、错误重试,共用一套「喂回 session」的机制。

=== "M3 上下文+可观测"

    长任务不撑爆(压缩是纯变换,不破坏恢复)+ 运行痕迹结构化(RunResult)+ 可选导出 OTel。

=== "M4 扩展层"

    一套洋葱中间件撑起所有扩展。环绕单步、栈每步重建 → 和「中断/恢复」天然兼容。连权限都重构成内置中间件,loop 里不再有特例。

=== "M5 脚手架"

    一条命令起项目。`rein new` 生成极简可跑起点(不堆空目录),`rein dev` 热重载。

---

## 接下来(都非必须)

框架已收官。可选方向:

- 发布打包到 PyPI
- 更多模板 / 内置中间件
- 文档站持续完善
- 真实场景打磨
