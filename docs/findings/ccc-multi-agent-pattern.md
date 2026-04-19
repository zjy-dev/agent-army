# Finding: Carlini 的 C Compiler 项目属于哪种多 Agent 模式？

> **日期**：2026-04-19
> **触发**：阅读 [Multi-agent coordination patterns](https://claude.com/blog/multi-agent-coordination-patterns) 后，尝试把 `docs/refs/parallel-claude-c-compiler/` 里记录的 Anthropic C Compiler 项目归入五模式谱系。
> **对话形态**：经过一次**初判 → 质疑 → 修正**的过程，最终结论比初判更干净、更极端。

---

## 🎯 最终结论

**纯 Shared State 模式，每个 session 内部嵌套 Generator-Verifier；Agent Teams 被明确否决。**

一句话定位：

> **"以 Git 仓库为 Shared State、以冷启动的 Ralph-loop 容器为可替换 worker、每个 session 内部跑 Generator-Verifier" 的两层混合架构——刻意避开了 Orchestrator、Message Bus、以及 Agent Teams 的持久记忆假设。**

---

## 🧭 推理过程

### 初判（不完全对）

第一反应是把它归为 **Agent Teams + Shared State 混合**，理由：

- 16 个长存 Docker 容器 → 像 Agent Teams 的 "worker persistence"
- Git 仓库被所有 agent 共享读写 → 像 Shared State
- 异质化角色（builder / critic / dedup / docs）→ 像 Agent Teams 的"专精化工人"

### 质疑点（用户指出）

> "但是每个 agent team member 其实是没有持久化的记忆的，每次拿任务都是新上下文吧。"

这一问直接戳中了**容器长存 ≠ teammate 长存**的概念混淆。

### 回到原文定义

文章对 Agent Teams 的核心判据：

> "Teammates stay alive across many assignments, **accumulating context and domain specialization** that improve their performance over time."

Agent Teams 的本质不是进程寿命，而是**上下文跨任务累积、专精随时间增长**。

### Carlini 项目的实际做法

```bash
while true; do
    claude -p "$(cat AGENT_PROMPT.md)" ...   ← 每次冷启动
done
```

- 每次 `claude` 调用都是 **fresh context**
- 上一次 session 的 in-memory 推理、试过的失败路径、形成的直觉——**全部丢失**
- "Ralph 信条" 第一条就是 **Fresh Context Is Reliability**（`harness-engineering/README.md:177`），这是**设计上的反向选择**

### 修正后的结论

Teammate 的持久记忆在这个项目里**是刻意缺席的**，不是遗漏。容器只是进程壳子，**teammate 的认知状态在每次 session 之间被主动清空**。

所以 Agent Teams 的特征**不成立**——只剩 Shared State。

---

## 🔑 关键概念转移：记忆在哪里？

既然单个 agent 没有持久记忆，整个系统的"记忆"存在哪？

**全部存在 Shared Store（git repo）里，以外化工件的形式**：

| 记忆类型 | 在哪里持久化 |
|---|---|
| 当前做什么 | `current_tasks/*.txt` |
| 试过哪些失败方法 | agent 维护的进度文档（博客原话："a running document with failed approaches"） |
| 项目整体状态 | README / 架构文档 |
| 代码本身的演化 | git history |
| 错误模式 | 日志文件 + CI 失败记录 |
| Agent 身份和归因 | `agent_${COMMIT}.log` 命名约定 |
| 角色差异化 | 每个容器挂载不同的 `AGENT_PROMPT.md` |

这正是 [`harness-essence.md:15`](../refs/harness-essence/harness-essence.md) 说的：

> "记忆的工程化含义不是'多存点东西'，而是把踩坑与验证过的结论**自动提炼成规则沉淀下来**。"

**Shared Store 承担了 agent 缺失的长期记忆。** 这不是设计不完善，这就是 Shared State 模式的正统形态——**去中心化 + 外部化记忆**。

---

## 📐 两种模式的记忆架构对比

| 维度 | Agent Teams（文章定义） | Carlini 的 CCC 项目 |
|---|---|---|
| 记忆载体 | Agent 自身的 context window | 外部 shared store（git repo） |
| 记忆生命周期 | 跨任务累积 | 单 session 内有效，结束即清零 |
| 专精形成 | Agent 逐渐熟悉自己的领域 | 无专精；角色通过 prompt + 外化痕迹维持 |
| 新 agent 加入成本 | 需要重新积累上下文 | 零成本——读 README + `current_tasks/` 即上手 |
| 单 worker 故障影响 | 丢失该 worker 的累积专精 | 无影响（反正要被替换） |

---

## 💡 一个反直觉的发现：角色的"人格"不在 agent 里

既然每次 session 都是 fresh context，**差异化角色（builder vs critic vs dedup）是靠什么维持的？**

只有两个途径：

1. **不同的 `AGENT_PROMPT.md`**（每个角色一个容器，每个容器挂不同 prompt）
2. **Shared store 里的角色痕迹**（比如 critic 写的 `reviews/`、dedup 写的 `refactor_notes/`）

**→ 角色的"人格"不在 agent 里，而在 shared state 里。**

Agent 本身是无状态的可替换单元；**人格 = prompt + 它在仓库里留下的历史痕迹**。

这个观察比初判更干净、更极端，也更符合 harness engineering 的原教旨：**工程重心从实现细节迁移到边界、接口、约束、断言与验收标准**。

---

## 📦 为什么这套设计可能比纯 Agent Teams 更鲁棒

Carlini 放弃"持久 teammate"这个假设，获得了：

- **任何 agent 崩溃、替换、扩容，系统不受影响**——因为没人持有独占的未外化记忆
- **新 agent 上手零成本**——读仓库即可，不需要"加入团队"流程
- **角色可以瞬时切换**——换个 prompt，同一容器就是另一个角色
- **调试友好**——所有状态都在 git history 里，可 bisection

代价是：**每次 session 都要重新读仓库、rebuild 上下文**，token 开销显著。`parallel-claude-c-compiler/01-overview.md:20-28` 的 $20k API 成本里，相当一部分就花在这个"不断重新理解项目"上。

**这是一个"用 token 换鲁棒性"的交易。**

---

## 🧩 嵌套 Generator-Verifier 层

虽然外层是 Shared State，**单个 agent 的 Ralph-loop 内部是标准的 Generator-Verifier**：

```
外层：Shared State（Git repo 作为唯一真相源 + 外化记忆）
       ├─ 并发原语：git merge 冲突 + current_tasks/ 锁文件
       ├─ 差异化：通过不同 AGENT_PROMPT.md 实现角色分化
       │         （这是 prompt 层面的异质化，不是 teammate 层面的）
       └─ 内层：每个 session 内部是 Generator-Verifier
                 ├─ Generator: Claude 写代码
                 └─ Verifier: CI / GCC torture test / kernel build
                    + grep-friendly 日志（[ERROR] <file>:<line>）
```

这印证了文章结尾的话："**production systems often combine patterns**"。

---

## ⚠️ 按文章标准，这套设计的结构性弱点

文章列举 Shared State 的失败模式，全部在这个项目中真实出现：

| 文章警告 | CCC 项目中的对应现象 |
|---|---|
| **重复工作** | "LLM 写代码常重复造轮子" → 专设 Deduplicator 角色应对（`03-design-lessons.md:126`） |
| **并发写冲突** | 通过 `current_tasks/` 锁文件 + git merge 冲突兜底 |
| **Reactive loop（token 空烧）** | "新 feature/bugfix 经常打破已有功能" + "16 个 agent 并行 ≈ 1 个 agent 效果"（单体任务场景） |
| **涌现行为难预测** | Claude 不小心 `pkill -9 bash` 自杀事件（`04-limitations-and-risks.md:70`） |
| **终止条件** | ✅ 处理得好：硬预算（$20k）+ 外部 oracle（GCC torture test 99%）+ 人类 kill switch |

**终止条件这一项做得足够好，是这个项目没陷入 reactive loop 失控的关键**——这恰好验证了文章的警告。

---

## 🪞 对 agent-army 的直接启示

### 1. 不要被"16 个并行 agent"的表象误导

真正的模式选择，不看**并行度**，看**记忆归属**：
- 记忆归 agent → Agent Teams
- 记忆归 store → Shared State

### 2. OpenCode subagent 默认是 Orchestrator-Subagent

不是 Agent Teams。每个 subagent 调用都是独立 context，完成即返回。想做 Shared State，需要显式把状态外化到仓库。

### 3. Ralph-loop 本质是 Shared State 的单元

如果 agent-army 采纳 Ralph-loop 作为长运行模式，就是在选 Shared State——必须同步接受它的全部设计义务：

- **终止条件前置设计**（不是 `while true` 就完事）
- **Reactive loop 监控**（token 消耗曲线 + 收敛指标）
- **记忆外化规范**（README / progress 文档 / 锁文件的约定）

### 4. 异质化靠 prompt + 仓库痕迹，不靠 agent 本身

想要"critic agent"和"implementer agent"的差异化，方案是：
- 两份不同的 prompt 文件
- 两份不同的写回路径（`reviews/` vs `src/`）
- **而不是**两个"长期训练出来"的 agent

---

## 📌 修正记录

- **2026-04-19 初判**：Agent Teams + Shared State 混合 ❌
- **2026-04-19 修正**：纯 Shared State + 嵌套 Generator-Verifier ✅
- **触发者**：用户指出 "每个 agent team member 其实是没有持久化的记忆"

**教训**：归类多 agent 模式时，不要被表层特征（进程数、容器寿命、角色名字）误导，要回到文章定义的**判据本质**——这次是**上下文是否跨任务累积**。
