# 常见问题

## Rein 和 LangChain 有什么不同?

定位不同:LangChain 是「大而全的生态」,Rein 是「极薄的单-agent harness」。落到**多轮会话封装**上,差别很典型:

> **LangChain** 内置几十种现成的会话存储后端、自动注入/回写历史 —— 开箱即用、生态厚。
> **Rein** 用一个可序列化的 `Session` + 只有 `save/load` 的 `SessionStore` —— 极薄、透明、状态完整,但数据库后端要自己写几行。

### LangChain 的会话封装(演进过三代)

1. `Memory` 类(早期,已弃用):`ConversationBufferMemory` 等,挂在 Chain 上自动管历史。
2. `RunnableWithMessageHistory`(LCEL):配 `get_session_history(session_id)` 回调,内置 `RedisChatMessageHistory` / `SQLChatMessageHistory` / Postgres / Mongo… 一堆。
3. LangGraph `checkpointer`(现在主推):`SqliteSaver` / `PostgresSaver` 按 `thread_id` 持久化整个 graph state。

```python
# LangChain:内置后端,一行就用,历史自动注入/回写(你看不见也不用管)
chain_with_history = RunnableWithMessageHistory(
    chain, lambda sid: RedisChatMessageHistory(sid, url="redis://..."),
    input_messages_key="input", history_messages_key="history",
)
```

### Rein 的会话封装

```python
# Rein:显式 读 → 跑 → 写;store 接数据库自己写几行
chat = agent.chat(session=store.load("u42"))
result = chat.send("你好")
store.save("u42", chat.session)
```

### 对比

| 维度 | LangChain | Rein |
|---|---|---|
| 内置存储后端 | 几十种(Redis/Postgres/Mongo/DynamoDB…)开箱即用 | 只 Memory/File;数据库自己写几行 |
| 抽象层数 | 多,演进三代(Memory→LCEL→LangGraph),易踩弃用/版本坑 | 一个 `Session` + `save/load` |
| 会话存什么 | 早期只存**消息历史**;LangGraph 才存整个 state | 存**整个运行状态**(消息+阶段+待办工具+用量+熔断) |
| 控制粒度 | 自动注入/回写(方便但黑盒) | 显式 load→run→save,状态全透明可序列化 |
| 依赖 | 重(core + 各 integration 包) | 核心只 pydantic+anyio |

### 一个深层差异

LangChain 早期的 message history **只存一串消息**。Rein 的 `Session` 存的是**整个运行状态** —— 包括「工具执行到哪、有哪些待批准的工具、熔断计数」。

所以在 Rein 里,**「多轮会话」和「HITL 审批 / 断点续跑」是同一套机制**:一个危险操作审批到一半,把 Session 存数据库,关机,改天读回来 `resume` 接着跑。这是「可序列化单步状态机」地基带来的 —— 会话状态本来就完整、可断点。

### 怎么选(客观)

- 要**开箱即用**、要现成的几十种存储后端、不在乎依赖重 → LangChain 生态是真优势。
- 要**极薄/可控/透明**、会话状态完整可断点、接什么数据库自己说了算 → Rein。

---

## 多轮会话到底怎么做?

会话的本体是 **`Session`**(不是 `Chat`)。`Chat` 只是个方便句柄;`Session` 能 `model_dump_json()` 存、`model_validate_json()` 还原。

- **单进程 / 脚本** → `agent.chat()` 内存版,最省事。
- **持久化 / 企业级 Web 服务** → `agent.chat(session=...)` + 你的 `SessionStore`:

```python
def handle(conv_id, user_msg):
    chat = agent.chat(session=store.load(conv_id))   # 从库读(没有=新会话)
    result = chat.send(user_msg)
    store.save(conv_id, chat.session)                # 写回库
    return result.output
```

一个 `Agent` 实例无状态、并发服务所有用户。详见 [人工审批与断点续跑](guides/resume.md) 与 [持久化参考](reference/persistence.md)。

---

## 会话能存数据库吗?

能。`Session` 就是一段 JSON,存哪都行 —— SQLite / PostgreSQL / MySQL / Redis / MongoDB 都可以。框架对数据库**零依赖、零绑定**,你只要实现 `SessionStore` 的 `save` / `load` 两个方法。完整范例(含 SQLite / Postgres / Redis)见 [持久化参考 · 存数据库](reference/persistence.md#存数据库)。

---

## 怎么接知识库(RAG)?

框架**不内置**向量库/embedding(极薄),但接法很自然,两种姿势:

- **检索即工具**(推荐,agentic):`@agent.tool def search_kb(query)`,模型自己决定何时检索。
- **`@agent.before_model` 钩子**:每轮自动检索并注入上下文。

向量库/embedding 用你自己的(chroma / pgvector / qdrant…)。详见 [中间件与扩展](guides/extensions.md) 与 [写工具](guides/tools.md)。

---

## 能做多 agent / AtoA 协作吗?

能,但方式是「组合」而非「内置编排」。Rein 是单-agent harness,不内置 CrewAI/AutoGen/LangGraph 那种编排引擎,但:

- **同进程协作** → 用 [agent 即工具](guides/multi-agent.md):委派 / 流水线 / `asyncio.gather` 并行。
- **跨进程 / 跨框架** → 把 agent [暴露成 A2A 服务](guides/a2a.md)(`serve_a2a` 一行),别人能发现+调用你,你也能反向调别人的 A2A agent。

每个子 agent 仍是完整 Rein agent(各自有熔断/审批/恢复/可观测)。复杂 agent 网络(动态路由、群聊)则建议用专门框架。详见 [多 agent 协作](guides/multi-agent.md)。

---

## 什么时候该用 Rein?

适合:**单 agent** 的工具型应用、要**可控/可观测/可恢复**的生产部署、在乎**依赖轻**、想要透明可断点的会话状态。

不适合:需要**多 agent 编排**、需要现成的 **RAG / 文档加载大生态**、想要开箱即用的几十种集成 —— 那些场景 LangChain / LlamaIndex 的生态更省事。
