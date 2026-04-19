# Finding: Anthropic 官方 Harness 属于哪种多 Agent 模式？

> **日期**：2026-04-19
> **触发**：读完 [`refs/long-running-harness/`](../refs/long-running-harness/)（Opus 4.6 时代的最终架构）后，把它放回 [`refs/multi-agent-coordination-patterns/`](../refs/multi-agent-coordination-patterns/) 的五模式谱系做归类。
> **对照对象**：上一轮对 Carlini C Compiler 的归类（见 [`ccc-multi-agent-pattern.md`](./ccc-multi-agent-pattern.md)）。

---

## 🎯 最终结论

**Orchestrator-Subagent 外层 + 嵌套 Generator-Verifier 的混合模式，完全不触及 Agent Teams / Message Bus / Shared State。**

精确定位：

> **Planner 作为 Orchestrator 派发 spec，Generator 作为 Subagent 执行，Evaluator 作为嵌套 Generator-Verifier 循环中的 Verifier 角色。整个系统是层级化、有明确协调者、低并发度的。**

---

## 🧭 推理过程

### 第一步：候选模式筛选

文章五模式按判据逐个对照：

#### ❌ Agent Teams — 不符合
文章定义：
> "Teammates stay alive across many assignments, **accumulating context and domain specialization** that improve their performance over time."

Anthropic 架构里：
- 每次 run 的 agent 都是**按角色一次性起**，run 结束即终止
- 角色间没有"跨任务累积的专精"
- Generator 在 run 内长 session，但**不跨 run 存活**
- Planner 做完一次 spec 后就退出

**不符合**"上下文跨任务累积"的核心判据。

#### ❌ Message Bus — 不符合
- 没有 publish/subscribe
- 没有 topic router
- 文件交接是**点对点**的（Planner→Generator, Generator→Evaluator），不是发布到总线

#### ❌ Shared State — 不符合
- 有明确的协调者（Planner 决定 spec，Evaluator 决定通过与否）
- 有层级关系（Planner 的输出是 Generator 的输入，不是平等读写）
- 虽然用文件通信，但**文件是定向工件，不是共享知识库**
- 没有"多个 agent 直接读写同一个 store 并反应彼此发现"的机制

#### ✅ Orchestrator-Subagent — 匹配
文章定义：
> "A lead agent receives a task and determines how to approach it. It may handle some subtasks directly while dispatching others to subagents. Subagents complete their work and return results, which the orchestrator synthesizes into a final output."

对照：
- **Planner = Orchestrator**：接收 raw prompt，扩成 spec，决定后续流程
- **Generator = Subagent**：接受派发，执行 bounded 任务（虽然 bounded 比典型场景大），返回产物
- **Evaluator = 特殊 Subagent**：接受派发，执行评估，返回结论
- Planner/Generator/Evaluator 都是**短命的**（每次 run 起一次）
- 信息流是**层级化**的

#### ✅ Generator-Verifier — 匹配（作为嵌套层）
文章定义的核心机制：
> "A generator receives a task and produces an initial output, which it passes to a verifier for evaluation. The verifier checks whether the output meets the required criteria and either accepts it as complete or rejects it with feedback."

**完全就是 Anthropic 架构的内层循环**：
- Generator 产出 app
- Evaluator 按 criteria 打分
- FAIL → Generator 修 → 再评
- 多轮直到通过或用尽预算（DAW 跑了 3 轮）

---

### 第二步：确认是混合而非单一

| 层次 | 模式 | 角色 |
|---|---|---|
| **外层** | Orchestrator-Subagent | Planner 派发 + Generator 执行 + Evaluator 评估 |
| **内层（Generator ↔ Evaluator）** | Generator-Verifier | 多轮 review-feedback-revise 循环 |

文章原话印证混合是常态：
> "A common hybrid uses **orchestrator-subagent for the overall workflow with shared state for a collaboration-heavy subtask**. Another uses message bus for event routing with agent team-style workers handling each event type. These patterns are building blocks, not mutually exclusive choices."

Anthropic 架构是 **"Orchestrator-Subagent + 嵌套 Generator-Verifier"** 这个组合，**不是**文章特别命名的那几个组合之一，但完全符合"building blocks"的拼装思路。

---

## 🔍 细节验证

### 验证 1：Planner 真的是 Orchestrator 吗？

Orchestrator 的定义特征：
- [x] **接收原始任务并决定如何拆解**——✅ Planner 从 1 句话扩到完整 spec
- [x] **派发子任务给不同 agent**——✅ spec 文件就是给 Generator 的派发
- [x] **综合结果**——⚠️ 这里不完全匹配——Planner **不综合**，Evaluator 才做最终判定

**→ 严格说，Planner 是"前置 Orchestrator"，最终综合由 Evaluator 承担。** 这是和典型 Orchestrator-Subagent 的一个细微差别。

### 验证 2：为什么不是 Agent Teams？

关键分水岭还是**记忆**：
- Generator 在单 run 内跑长 session（Opus 4.6 可 2+ 小时）——**这是 session 内部的连贯**
- 但 run 结束即终止——**不跨 run 累积**
- 下一次新任务来时，起一个全新的 Generator——没有"这个 Generator 之前在别的任务里学到了什么"

这和 Agent Teams 定义的"teammate 跨多次任务累积专精"有本质区别。

**Anthropic 架构的 Generator 更像 Claude Code 里的 subagent——bounded 任务内可以很长、很深，但生命周期就是这一次任务。**

### 验证 3：为什么文件通信不等于 Shared State？

两者的关键区别：

| 维度 | Shared State | Anthropic 架构的文件 |
|---|---|---|
| 读写模式 | **任意 agent 任意时刻读写同一文件** | 定向交接：A 写 → B 读 |
| 协调机制 | 无中央协调，靠读写冲突兜底 | 有明确的 orchestrator 决定顺序 |
| 文件生命周期 | 持续演进的知识库 | 一次性工件（spec/report） |
| Agent 彼此可见性 | 所有人能看到所有人的发现 | 只看自己被派发到的输入 |

**→ 文件只是通信媒介，不等于 Shared State。** 关键看**语义**：是层级派发还是对等共享。

---

## 🧩 和 Carlini C Compiler 的对照表

这是两次归类结论的并排对比：

| 维度 | Carlini CCC | Anthropic 最终版 |
|---|---|---|
| **外层模式** | Shared State | Orchestrator-Subagent |
| **内层模式** | Generator-Verifier（单 agent 内部 Ralph-loop）| Generator-Verifier（跨 agent 的外层循环）|
| **是否 Agent Teams** | ❌ 否（冷启动，无持久记忆）| ❌ 否（run 内长 session，但不跨 run）|
| **协调者** | 无 | Planner + Evaluator |
| **agent 并发度** | 16 | 1-3（串行为主）|
| **记忆外化程度** | 极高（全部在 git）| 中（靠 SDK auto-compaction + 文件交接）|
| **任务特性** | 有 oracle + 可并行 | 无 oracle + 需完整产品 |
| **终止条件** | 预算 + 人工 + oracle 通过率 | Evaluator 多轮后无显著新问题 |

两个架构都是**混合模式**，但混合的**基础成分不同**。

---

## 🧠 更深一层：为什么两个架构都选了 Generator-Verifier 作为内层

这是一个很有意思的收敛点。

### Carlini 的 Generator-Verifier
- Generator = Claude 写编译器代码
- Verifier = GCC torture test / Linux kernel build / Doom 跑通
- 位置：**单个 Ralph-loop agent 的内部**

### Anthropic 的 Generator-Verifier
- Generator = Generator agent 实现 app
- Verifier = Evaluator agent 用 Playwright 点击
- 位置：**多 agent 外层循环**

### 共同点
**两者都认识到：没有独立 verification 机制，agent 会自欺**。

区别只是 verifier 的**身份**：
- Carlini 用**外部工具**（GCC、torture test）当 verifier
- Anthropic 用**另一个 LLM agent**（带 Playwright）当 verifier

**→ Generator-Verifier 是两个架构在"如何抵抗 LLM 宽容倾向"这个问题上的共同答案。** 这也印证了 `multi-agent-coordination-patterns/03-key-insights.md` 第 1 条："Evaluator 的质量 = 系统上限"。

---

## 💡 意外发现：Sprint Contract 曾经想做什么

后作早期版本用过 Sprint + Sprint Contract，后来被砍。

**在五模式谱系里回看**：Sprint Contract **其实是在尝试把 Orchestrator-Subagent 变成更细粒度的 Generator-Verifier**——
- Generator 提议 contract（"我要做这些，这样验证"）
- Evaluator 审 contract
- 两方谈判到一致
- 这其实是一轮"**预注册的 Generator-Verifier**"——在写代码之前先把 verifier 的判据写死

Opus 4.6 之后 Sprint 被砍，意味着：**预注册 criteria 这件事可以放到更粗粒度（run 级别）去做，不需要 sprint 级别的频率**。

**这是一个 harness 组件从"频繁运行的细粒度 Generator-Verifier"退化到"粗粒度 Generator-Verifier"的典型案例。** 如果模型进一步进步，可能连粗粒度 Evaluator 都可以退化到"只在最后一次跑一次"。

---

## 🪞 对 agent-army 的直接启示

### 1. OpenCode 的天然模式就是 Orchestrator-Subagent

OpenCode 主 agent + Task tool 派子 agent 是**原生就是 Orchestrator-Subagent**——这和 Anthropic 最终架构**同构**。

**含义**：如果 agent-army 要实现类似 Anthropic 的三 agent 架构，**不需要自建调度层**，直接用 OpenCode 的 subagent 调用机制即可：
- 主 agent 扮演 Planner + 外层编排
- 派 subagent 扮演 Generator（实现）
- 派 subagent 扮演 Evaluator（审查）

### 2. Generator-Verifier 的嵌套位置要慎重选

Carlini 把 Generator-Verifier 放在单 agent 内部（Ralph-loop），Anthropic 放在多 agent 外层。

对 agent-army：
- **单 agent 任务**：内嵌 Generator-Verifier（让 agent 自己跑测试修 bug）
- **多 agent 任务**：外置 Generator-Verifier（独立 critic subagent）
- **两者都上**：嵌套双层，Carlini 式的外加 Anthropic 式的

### 3. "记忆外化 vs SDK 管理"是架构选型的首个分岔

在动手前先问：
- 任务能在单次 session 做完吗？能 → Anthropic 路线
- 不能 → 是否有天然 oracle？有 → Carlini 路线；没有 → 仍走 Anthropic + 多轮 review
- 需要 16x 的并发吗？需要 → Carlini 路线；不需要 → Anthropic 路线

### 4. 不要混淆"文件通信"和"Shared State"

用文件通信是良好工程实践，**但不等于选了 Shared State 模式**。
- Shared State 的核心是**去中心化 + 任意读写**
- Orchestrator-Subagent 也用文件，**但是定向交接**

agent-army 里约定工件格式时要说清楚：**这是 spec（单向派发）还是 shared knowledge（任意读写）**。语义不同，并发控制、终止条件、失败模式都不同。

### 5. Evaluator 永远先于 Generator 设计

两个架构都证明：**"没有好 verifier，系统就在高效朝错方向飞奔"**（harness-essence.md:19 的原话）。

agent-army 设计任何新 subagent 时的标准问题：
- 这个 subagent 的产出由谁验证？
- Verifier 是独立工具、独立 agent、还是自我评估？
- 自我评估需要什么保护措施（Goodhart 对抗）？

---

## 📌 一句话归纳

> **Anthropic 的最终 harness 是"Orchestrator-Subagent 外层 + 嵌套 Generator-Verifier"——Planner 派发、Generator 执行、Evaluator 审查，agent 短命但 session 内可长跑，文件定向交接而非共享读写，模型自身的长 session 连贯性让 Agent Teams 和 Shared State 都不必要。**
>
> **和 Carlini 的 Shared State + 嵌套 Generator-Verifier 互为镜像：任务特性不同 → 选的外层模式不同 → 但内层的 Generator-Verifier 这个质量闭环收敛到同一个答案。**

---

## 📌 修正记录

对比上一轮 `ccc-multi-agent-pattern.md` 的经历：

- 上一轮的教训："不要被表层特征（进程数、容器寿命）误导，要回到判据本质"
- 这一轮的新教训："**不要被通信媒介（都用文件）误导，要回到语义本质**——定向派发 vs 共享读写是不同模式"

---

## 🔗 相关文档

- [`refs/long-running-harness/`](../refs/long-running-harness/) — 官方最终架构细节
- [`refs/multi-agent-coordination-patterns/`](../refs/multi-agent-coordination-patterns/) — 五模式谱系
- [`findings/ccc-multi-agent-pattern.md`](./ccc-multi-agent-pattern.md) — CCC 的模式归类（Shared State）
- [`findings/ccc-and-harness-synthesis.md`](./ccc-and-harness-synthesis.md) — 两路线的合观与选型
