# 持久化 · 上下文压缩

`SessionStore`(存会话)与 `CompactionStrategy`(压缩历史)。教程见 [断点续跑](../guides/resume.md)、[上下文与可观测](../guides/context.md)。

---

## SessionStore(协议)

会话存储的最小接口。redis / 数据库自己实现这俩方法即可(鸭子类型)。

```python
def save(self, id: str, session: Session) -> None
def load(self, id: str) -> Session | None
```

---

## MemorySessionStore

进程内字典存储(测试 / 单进程)。存 JSON 串,语义和文件/数据库一致。

```python
MemorySessionStore()
```

| 方法 | 说明 |
|---|---|
| `save(id, session)` | 保存(覆盖同 id) |
| `load(id) -> Session \| None` | 取回,不存在返回 None |
| `delete(id)` | 删除 |
| `"id" in store` | 是否存在 |

```python
from rein import MemorySessionStore
store = MemorySessionStore()
store.save("task-1", result.session)
session = store.load("task-1")
```

---

## FileSessionStore

把每个会话存成 `{目录}/{id}.json`。无额外依赖,适合本地长任务续跑。

```python
FileSessionStore(directory: str | Path)
```

| 方法 | 说明 |
|---|---|
| `save(id, session)` | 写文件 |
| `load(id)` | 读文件,不存在返回 None |
| `delete(id)` | 删文件 |

```python
from rein import FileSessionStore
store = FileSessionStore("./sessions")
store.save("task-1", result.session)     # → ./sessions/task-1.json
```

非法 id(含 `/`、`..` 等)会报 `ValueError`,防止写到目录外。

---

## 存数据库

框架只内置 Memory/File,**数据库后端自己写** —— 只要实现 `save` / `load` 两个方法(鸭子类型,无需继承)。核心就两件事:存的时候 `session.model_dump_json()`,读的时候 `Session.model_validate_json(...)`。

### SQLite(标准库,零额外依赖)

```python
import sqlite3
from rein import Session

class SQLiteSessionStore:
    def __init__(self, path: str):
        self.path = path
        with sqlite3.connect(path) as c:
            c.execute("CREATE TABLE IF NOT EXISTS sessions (id TEXT PRIMARY KEY, data TEXT)")

    def save(self, id: str, session: Session) -> None:
        with sqlite3.connect(self.path) as c:
            c.execute("INSERT OR REPLACE INTO sessions VALUES (?, ?)",
                      (id, session.model_dump_json()))

    def load(self, id: str) -> Session | None:
        with sqlite3.connect(self.path) as c:
            row = c.execute("SELECT data FROM sessions WHERE id=?", (id,)).fetchone()
        return Session.model_validate_json(row[0]) if row else None
```

### PostgreSQL(用 psycopg)

```python
import psycopg
from rein import Session

class PostgresSessionStore:
    def __init__(self, dsn: str):
        self.dsn = dsn
        with psycopg.connect(dsn) as conn:
            conn.execute("CREATE TABLE IF NOT EXISTS sessions (id TEXT PRIMARY KEY, data JSONB)")

    def save(self, id: str, session: Session) -> None:
        with psycopg.connect(self.dsn) as conn:
            conn.execute(
                "INSERT INTO sessions (id, data) VALUES (%s, %s) "
                "ON CONFLICT (id) DO UPDATE SET data = EXCLUDED.data",
                (id, session.model_dump_json()),
            )

    def load(self, id: str) -> Session | None:
        with psycopg.connect(self.dsn) as conn:
            row = conn.execute("SELECT data FROM sessions WHERE id=%s", (id,)).fetchone()
        return Session.model_validate_json(row[0]) if row else None
```

### Redis(用 redis-py)

```python
import redis
from rein import Session

class RedisSessionStore:
    def __init__(self, client: redis.Redis, prefix="rein:sess:", ttl=None):
        self.client, self.prefix, self.ttl = client, prefix, ttl

    def save(self, id: str, session: Session) -> None:
        self.client.set(self.prefix + id, session.model_dump_json(), ex=self.ttl)

    def load(self, id: str) -> Session | None:
        raw = self.client.get(self.prefix + id)
        return Session.model_validate_json(raw) if raw else None
```

!!! tip "用法都一样"
    无论哪种后端,业务侧都是同一套三行:
    ```python
    chat = agent.chat(session=store.load(conv_id))
    result = chat.send(user_msg)
    store.save(conv_id, chat.session)
    ```
    换数据库 = 换 store,业务代码一行不动。可跑示例见仓库 `examples/enterprise_chat.py`。

---

## CompactionStrategy(协议)

上下文压缩策略。输入消息列表、输出(可能更短的)消息列表 —— **纯变换**,不破坏可序列化/恢复。

```python
async def compact(self, messages: list[Message]) -> list[Message]
```

没超过自身阈值时应原样返回。`Agent(compaction=...)` 注入后,loop 每次问模型前自动调一次。

---

## SlidingWindow

滑动窗口:只保留 system + 最近 N 条,丢更早的。

```python
SlidingWindow(max_messages: int, *, keep_system: bool = True)
```

| 参数 | 说明 |
|---|---|
| `max_messages` | 保留的「最近非 system 消息」条数上限 |
| `keep_system` | 是否始终保留 system 提示(默认 True) |

会自动去掉裁剪后「开头的孤儿 tool 消息」(它对应的 assistant 调用被裁掉了)。

```python
from rein import Agent, SlidingWindow
agent = Agent("anthropic/...", compaction=SlidingWindow(max_messages=20))
```

---

## SummarizeCompaction

摘要式:超 token 阈值时,把旧历史折叠成一条摘要,保留近期原文。

```python
SummarizeCompaction(max_tokens: int, *, keep_recent: int = 4, summarizer=None)
```

| 参数 | 说明 |
|---|---|
| `max_tokens` | 触发压缩的估算 token 阈值 |
| `keep_recent` | 保留最近多少条原文不折叠(默认 4) |
| `summarizer` | `None` → 机械摘要(拼接截断,不联网,主要降条数);传有 `.complete` 的 Provider → LLM 真摘要;也可传 `async callable(messages)->str` |

```python
from rein import Agent, SummarizeCompaction
agent = Agent("anthropic/...", compaction=SummarizeCompaction(max_tokens=8000, keep_recent=6))
```

!!! tip
    机械摘要主要降「消息条数」;真正大幅降 token(旧历史很长时)请注入 Provider 做 LLM 摘要。

---

## estimate_tokens

```python
def estimate_tokens(messages: list[Message] | str) -> int
```

粗略估算 token 数(近似:非 ASCII≈1、ASCII≈0.25/char)。保守上界,用于「该不该压」的判断,不用于计费。

```python
from rein import estimate_tokens
estimate_tokens(session.messages)
```
