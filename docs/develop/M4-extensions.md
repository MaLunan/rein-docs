# M4 —— 扩展层

> **一句话目标**:提供一套与「可恢复 loop」兼容的扩展机制(中间件/钩子/事件),并补上 Docker 沙箱与插件发现。
> **前置依赖**:M2(关键!中间件必须在可恢复 loop 定稿后设计,才能满足兼容约束)
> **状态**:✅ 已完成(2026-06-27,本地单测全绿;中间件/钩子/事件/权限重构/DockerRuntime/插件 均落地)

---

## 一、本阶段要解决的需求

| 需求 | 在 M4 的体现 |
|---|---|
| 需求 6 生命周期与钩子 | 一套中间件(洋葱)+ 钩子语法糖 + 只读事件 |
| 需求 3 一键衔接 Runtime(沙箱) | `DockerRuntime` |
| (生态扩展) | 插件 entry points 自动发现 |

---

## 二、交付物 / 验收标准

1. ✅ `@agent.middleware` 洋葱中间件:可前后插逻辑、可短路、可统一 try/except。
2. ✅ 钩子语法糖(如 `@agent.before_tool`)= 中间件的便捷封装,内部统一实现。
3. ✅ 事件订阅(`agent.on(...)`)只读观测,不能改流程。
4. ✅ **权限即钩子**:`PermissionPolicy` 用 `before_tool` 拦截统一实现,Loop 里不再写权限特例。
5. ✅ `DockerRuntime` 在容器内执行工具/命令,与 LocalRuntime 接口一致,改配置即切换。
6. ✅ 第三方包经 entry points 被自动发现注册。
7. ✅ 扩展机制在「中断→恢复」场景下不破坏正确性(回归测试)。

---

## 三、开发注意点(坑与约束)⚠️

1. **兼容可恢复的硬约束(最重要)**:
   - 中断点**只允许在工具执行边界**,不允许在中间件中途暂停。
   - 中间件环绕的是**单步 `step`**,不是整个 loop;**中间件不得跨步持有状态**(要持有就放 Session)。
   - 恢复时中间件栈**按需重建**(它们是无状态环绕逻辑)。
2. **只暴露一种主心智**:对用户主推「中间件」;钩子是糖、事件是只读旁路。不要又搞出三套并列的重叠 API。
3. **Docker 沙箱安全**:资源限制、网络隔离、文件挂载范围都要保守默认;沙箱是为「跑 LLM 生成的代码」准备的。
4. **`docker` 依赖走 extras**,DockerRuntime 内部延迟 import。

---

## 四、TodoList

- [x] `Middleware` 调度引擎(`middleware.py`:洋葱 `dispatch`,环绕单步 step,支持短路)
- [x] 钩子语法糖:`@before_model`/`@after_model`/`@before_tool`/`@after_tool` → 统一转中间件
- [x] 事件总线 + `agent.on(event, handler)`(只读;"step"/"tool" 事件)
- [x] 权限重构为内置 `permission_middleware`(ask 产中断短路;loop.step 去权限特例;deny 仍在 Runtime)
- [x] `DockerRuntime`(延迟 import docker,getsource 容器执行,接口对齐 Local,保守默认)
- [x] 插件系统:`plugins.py` entry points(`rein.plugins` group)发现 + 加载
- [x] 兼容性回归:`test_extensions_compat`(中间件 + ask 中断/恢复 共存,栈按需重建)
- [x] 更新 README / document / handoff / code-guide / memory 进度

---

## 五、与设计文档的对应

- `DESIGN.md` §3 需求6、需求3;§4 自我否决 #4;§5.1 扩展层;§6 路线图 M4;§7 开放问题 #5。
- 设计哲学:机制(中间件调度引擎)= 核心;实例(你写的某个中间件)= 扩展。
