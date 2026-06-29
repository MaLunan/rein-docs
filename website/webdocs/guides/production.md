# 生产部署实践

把 Rein agent 放到生产环境前,这一页集中了**密钥、日志、熔断、错误处理、并发、部署**的最佳实践,以及一份上线检查清单。

## 1. 密钥管理(最重要)

- **key 走环境变量,框架不碰**:LiteLLM 从 `ANTHROPIC_API_KEY` / `OPENAI_API_KEY` 等环境变量读;Rein 自己**不读、不存、不传** key。
- **别把 key 写进代码或 prompt**:写进 message 会进入 `Session`,而 Session 可能被持久化(见 §6),等于把 key 落盘。
- **`.env` 必须 gitignore**:`rein new` 生成的项目**已自带 `.gitignore` 忽略 `.env`** —— 别手动把 `.env` 加进 git。
- **日志已脱敏**:`enable_logging` 不记 messages / 工具结果 / key,只记工具名、耗时、token。
- **异常原文注意**:provider / 工具异常的文本会进 `RunResult.interrupt` 和 `ToolResult`。LiteLLM 的异常不含 key(key 在 HTTP header,不在异常体),但若你的**自定义工具 / provider** 可能在异常里带敏感信息,用 `after_tool` 中间件脱敏后再回填。

## 2. 日志与可观测

```python
from rein import enable_logging
enable_logging("INFO", json=True)   # 单行 JSON,接 ELK / Loki / CloudWatch
```

- 每条日志带 `trace_id`,串起一次运行的全部动作(`run started` → `tool done` → `run finished`)。
- `RunResult.steps` 是**可回放的运行记录**(每步类型、摘要、耗时、token)。
- 分布式追踪:`pip install "rein-agent[otel]"` + `export_run(result)` 导出到 OpenTelemetry。

## 3. 熔断配置(防烧钱)

生产环境**四道闸都要显式设**:

```python
from rein import Agent, LoopConfig

agent = Agent(
    "anthropic/claude-opus-4-8",
    config=LoopConfig(
        max_iterations=50,      # 最多几轮「模型→工具」
        max_tokens=200_000,     # 累计 token 上限(成本闸)
        timeout_s=120,          # 墙钟超时
        detect_loops=True,      # 原地打转检测
    ),
)
```

任一触顶就安全停下,`RunResult.stop_reason` 会告诉你是哪道闸。

## 4. 错误处理与韧性

- **fallback 主备自动切换**:`Agent("anthropic/...", fallback=["openai/gpt-4o", "deepseek/deepseek-chat"])`,主模型限流/报错自动切下一个。
- **可重试错误**(限流/超时/5xx)→ 产出 `error` 中断态 → `agent.resume(session)` 重试;**致命错误**(鉴权/参数)直接抛。
- **工具异常不崩 loop**:工具内部异常会被封装成 `is_error` 结果回填,模型读到后自愈。

## 5. 并发与会话

- **Agent 是无状态蓝图** → 可作全局单例 / 多请求并发共享,不会串台。
- **每个请求 / 用户用独立 `Session`**:`agent.chat(session=...)` 或传入独立 session。
- **别跨请求共享 `Chat`**(它持有会话状态)。

## 6. 部署形态:无状态 Web 服务

Agent 全局单例 + 每请求从 `SessionStore` 取放会话:

```python
from rein import Agent
from rein import Session, FileSessionStore   # 或自实现 redis / 数据库 store

agent = Agent("anthropic/claude-opus-4-8")   # 全局单例
store = FileSessionStore("./sessions")

def handle(user_id: str, msg: str) -> str:
    session = store.load(user_id) or Session()
    chat = agent.chat(session=session)
    reply = chat.send(msg)
    store.save(user_id, chat.session)         # Session 可序列化,存哪都行
    return reply
```

- **长任务**:跑一半中断 → `store.save` 存盘 → 改天 `agent.resume` 接着跑。
- **持久化后端随你**:Session 是 `model_dump_json()` 可序列化的,文件 / Redis / Postgres 都行,框架零绑定。

## 7. 上线前检查清单

- [ ] API key 在环境变量里,`.env` 已被 gitignore
- [ ] 熔断四道闸(iterations / tokens / timeout / detect_loops)都配了
- [ ] 配了 `fallback` 备用模型
- [ ] `enable_logging` 接入了日志系统
- [ ] 并发场景:Agent 单例共享、每请求独立 Session
- [ ] 跑过目标厂商的真实连通性冒烟
- [ ] 自定义工具 / provider 的异常不会带出敏感信息
