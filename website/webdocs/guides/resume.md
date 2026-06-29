# 人工审批与断点续跑

Rein 的恢复机制建立在一条地基上:**暂停 = 把会话存盘,恢复 = 把会话喂回去**。同一套机制撑起人工审批(HITL)、长任务续跑、错误重试。

## 人工审批(HITL)

把权限设成 `ask`,工具执行前会**暂停**,产出一个中断态:

```python
from rein import Agent, LoopConfig

agent = Agent("anthropic/claude-opus-4-8", config=LoopConfig(permission="ask"))

@agent.tool
def delete_db(name: str) -> str:
    "删除数据库(危险)"
    return f"已删除 {name}"

result = agent.run("清理生产库")

if result.status == "interrupted":
    print(result.interrupt.message)        # "待批准执行工具:delete_db ..."
    # 你来决定:
    result = agent.resume(result.session, approve=False)   # 拒绝
    print(result.output)                   # 模型读到拒绝,改走安全做法(自愈)
```

- `approve=True` → 执行工具,继续跑。
- `approve=False` → 把「被拒绝」回填给模型,模型自己换个安全方式。

## CLI 里的交互式审批

```python
# 每遇审批就在终端问 y/N,自动续跑
result = agent.run_interactive("帮我清理目录")
```

它和服务端「产出中断态交给上层」用的是**同一套**机制,不是两套。

## 恢复的本质:喂回 session,不靠挂起的协程

`resume` **不是**「继续一个卡住的协程」,而是「拿一份 session 数据,从它的阶段接着推进」。所以中断态可以**存盘、关机、换台机器、明天再 resume**:

```python
from rein import FileSessionStore

store = FileSessionStore("./sessions")

result = agent.run("危险任务")
if result.status == "interrupted":
    store.save("task-42", result.session)   # 存盘,可以关机了

# ……改天,另一个进程……
session = store.load("task-42")             # 读盘
result = agent.resume(session, approve=True)  # 续跑到完成
```

## 持久化:SessionStore

```python
from rein import MemorySessionStore, FileSessionStore

MemorySessionStore()          # 进程内(测试/单进程)
FileSessionStore("./dir")     # 存成 {id}.json
```

接口只有 `save(id, session)` / `load(id)`。redis / 数据库是你的事 —— 实现这俩方法即可(鸭子类型,无需继承)。"持久化不绑后端"。

## 错误也能恢复重试

模型调用遇到**可重试错误**(限流 / 超时 / 5xx)时,会建模成 `error` 中断态,你可以 resume 重试:

```python
result = agent.run("...")
if result.status == "interrupted" and result.interrupt.type == "error":
    result = agent.resume(result.session)   # 重跑这一步
```

致命错误(鉴权 / 参数错)则直接抛 —— 重试也没用,抛出来最清晰。
