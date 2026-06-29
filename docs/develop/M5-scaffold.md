# M5 —— 脚手架

> **一句话目标**:提供「一键起项目」的极简脚手架与开发期增益,但坚决不堆重目录结构。
> **前置依赖**:M0
> **状态**:✅ 已完成(2026-06-27,本地单测全绿;`rein new`/`rein dev` 实跑通过)

---

## 一、本阶段要解决的需求

| 需求 | 在 M5 的体现 |
|---|---|
| 形态:脚手架 CLI + 运行时库 | `rein new` / `rein dev` |
| 需求 4 方便人用 | 给新手一个「能直接跑的起点」 |

---

## 二、交付物 / 验收标准

1. ✅ `rein new myagent` 生成**极简可跑**起点:一个 `main.py` + `.env` 模板 + 可选 `rein.toml`(**不生成一堆空目录**)。
2. ✅ `cd myagent && python main.py` 直接能跑(填好 key 后)。
3. ✅ `rein dev` 提供热重载 + 更详细的运行追踪输出。
4. ✅ 至少 1~2 个模板(minimal、coder 雏形)。

---

## 三、开发注意点(坑与约束)⚠️

1. **默认极简**:`rein new` 不要学 Django 生成层层目录;agent 项目通常就一个文件。模板用来给**可运行示范**,不是强制结构。
2. **砍掉 `rein run`**:`python main.py` 即可,不做多余命令。
3. **模板可运行优先**:模板里给的是「能跑通的最小例子」,不是占位骨架。
4. **CLI 用 Typer**,自动帮助;`dev` 的热重载基于文件监听重启。

---

## 四、TodoList

- [x] Typer CLI 入口(`rein`,`cli.py`;走 `rein-agent[cli]` extras + `[project.scripts]` entry point)
- [x] `rein new <name> [--template]`:极简起点生成(`scaffold.create_project` 纯函数 + CLI 门面)
- [x] 模板:`minimal`(5 行)、`coder`(read_file/run_shell 雏形 + permission=ask)
- [x] `rein dev`:标准库轮询 mtime 热重载 + 给子进程设 REIN_DEV=1(可挂追踪)
- [x] CLI 集成测试(`test_scaffold` 测纯函数;`test_cli` 用 CliRunner 测门面,生成的 main.py 可 compile)
- [ ] (可选)文档站骨架 —— 暂缓(非核心,后续按需)
- [x] 更新 README / document / handoff / code-guide / memory 进度

---

## 五、与设计文档的对应

- `DESIGN.md` §2 形态决策;§3 需求4;§4 自我否决 #3;§6 路线图 M5;§7 开放问题 #6(命名)。
