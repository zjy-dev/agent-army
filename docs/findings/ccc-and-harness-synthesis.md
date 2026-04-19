# Finding: Carlini C Compiler 与 Anthropic 官方 Harness 的合观

> **日期**：2026-04-19
> **触发**：连读三份材料后产生的跨案例综合观察——
> - [Claude C Compiler（Carlini, 2026-02）](../refs/parallel-claude-c-compiler/)
> - [Effective Harnesses for Long-running Agents（Young, 2025-11）](../refs/long-running-harness/)
> - [Harness Design for Long-Running App Development（Rajasekaran, 2026-03）](../refs/long-running-harness/)
>
> 这三份材料表面上讲不同的事，**实际上从三个截然不同的角度在回答同一个核心问题**：如何让 agent 在超出单 session 能力的任务上稳定推进。

---

## 🎯 核心发现

**三份材料代表了 Harness 工程的三种截然不同的哲学，每种都完整自洽，但基于根本不同的"关于模型的假设"。它们不是演进关系，是三条平行路线。**

| 维度 | Carlini C Compiler | Anthropic 官方（最终版） |
|---|---|---|
| **核心比喻** | 16 个健忘工人在同一工地 | 3 个分工明确的工程师（Planner / Coder / QA）|
| **记忆架构** | 完全外化到 git | Agent SDK auto-compaction 管 context |
| **Agent 寿命** | Session 冷启动，容器长存 | Session 长存（Opus 4.6 可 2+ 小时） |
| **并行度** | 高（16 并发）| 低（三个串行角色）|
| **协调机制** | Git merge + 锁文件，去中心化 | 文件交接，层级化 |
| **模式归类** | Shared State + 嵌套 Generator-Verifier | Orchestrator-Subagent + 嵌套 Generator-Verifier |
| **对模型的假设** | 模型会忘、会自杀、会跑偏——**全靠外部约束** | 模型越来越强——**定期 prune 脚手架** |
| **扩展性期待** | 加更多 agent、更多角色 | 等模型升级、砍掉 scaffolding |

---

## 🧩 第一层感悟：两种哲学的本质分歧

### Carlini 的底层信念
> **模型是不可靠的执行单元，系统可靠性必须由外部机制提供。**

具体表现：
- 每个 agent 每次 session **冷启动**——主动放弃持久记忆
- **记忆外化到 git**——所有状态用工件表达
- **Git merge 冲突作锁**——最少的协调基础设施
- **异质化通过 prompt + 仓库痕迹维持**——agent 本身是可替换的
- **终止条件靠人 + 预算 + oracle 测试**

这是一种**"工业化装配线"**哲学：每个工人的记忆可丢弃，工件（零件）在传送带（git）上流动，只有仓库是永恒的。

### Anthropic 的底层信念
> **模型是越来越能干的通用工程师，harness 的作用是临时补齐当前能力缺口。**

具体表现：
- **相信 session 内的连贯推理**——Opus 4.6 可连续 2+ 小时
- **相信 compaction**——SDK 自动管理 context
- **分工明确的角色**——Planner / Generator / Evaluator 各司其职
- **Harness 组件定期 prune**——每次模型升级审视一遍
- **终止条件靠 Evaluator 判定**——模型自己决定何时足够好

这是一种**"渐进式外包"**哲学：相信模型会越来越像真实工程师，harness 的工作是找到"当下差什么"并补上，随模型进步逐步收回补丁。

### 它们不是对立，而是对应不同任务
Carlini 的任务：**从零造 C 编译器**——高度专业、错误代价高、需要 100,000 行代码
Anthropic 的任务：**从 1 句话造 DAW**——全栈 web app、错误代价低、目标模糊但可迭代

**分歧点不在"谁对"，在于"任务和模型能力的比值在哪里"**——这正是后作强调的原则。

---

## 🧩 第二层感悟：记忆位置决定架构形态

三份材料里，**记忆存在哪**决定了剩下的一切设计选择。

| 方案 | 记忆位置 | 这个选择强制了什么 |
|---|---|---|
| **Carlini** | 全部在 Shared Store（git）| 必须去中心化；必须容忍重复；必须有强终止条件；必须 Docker 隔离 |
| **官方前作（Young）** | Artifacts（progress.txt + JSON + git）+ 每 session 重建 | 必须结构化工件；必须 bootstrap 三部曲；必须 JSON 不用 Markdown |
| **官方后作（Rajasekaran）** | **Agent 内部 context**（靠 auto-compaction）+ 文件交接 | 可以信任长 session；可以砍掉 reset；需要 Evaluator 补 self-eval 漏洞 |

### 一条推论

**记忆位置 = 对模型信任度的量化表达**。

- 完全信任模型内部记忆 → 单 agent 长 session（当前只能做简单短任务）
- 部分信任 → Agent SDK + Evaluator（当前官方最终版）
- 零信任 → 全部外化到 store（Carlini）

这是一个**连续谱**，不是离散选择。未来模型更强 → 信任度提升 → 外化程度降低。

---

## 🧩 第三层感悟：验证机制的三种形态

三份材料都有 verification，但形态差异巨大：

| 来源 | Verifier 形态 | 耦合度 |
|---|---|---|
| **Carlini** | 外部 oracle（GCC torture test + Linux kernel build + Doom）| 低耦合——oracle 客观、独立于 agent |
| **官方前作** | Coding Agent 自己用 Puppeteer 验（+feature list JSON 状态追踪）| 中耦合——但自评有 Goodhart 风险 |
| **官方后作** | 独立 Evaluator Agent + Playwright MCP + criteria 评分 | 中高耦合——但独立 LLM + 严格 prompt |

### 关键观察

**Carlini 的做法最硬核，也最依赖任务特性**：C 编译器恰好有几十年积累的 oracle 测试套件。大多数任务**没有这种 oracle**。

**官方后作的 Evaluator 是"在没有 oracle 的情况下最接近 oracle 的近似"**——独立 LLM + 严格 prompt + 真实工具（Playwright），在没有绝对裁判的情况下提供可接受的判定。

**→ 任务是否有 oracle，直接决定选哪条路线。**

| 任务是否有 oracle | 推荐路线 |
|---|---|
| **有**（已存测试套件、已知正确系统、形式化 spec）| Carlini 路线：多 agent 并行 + oracle bisection |
| **没有** | Anthropic 路线：独立 Evaluator + criteria 评分 |

---

## 🧩 第四层感悟：Harness 组件的"时代性"

Anthropic 后作提出的核心方法论：**每个组件都编码一个"关于模型无能的假设"，会 stale**。

**把这个框架套到 Carlini 项目上会有什么结论？**

Carlini 的 harness 组件对应假设：

| 组件 | 假设 | Opus 4.6+ 时代是否还成立？ |
|---|---|---|
| Docker 隔离 | 模型会 `pkill -9 bash` 自杀 | **基本仍成立**——安全需求不随能力消失 |
| 每 session 冷启动 | 模型会 context pollution | **部分失效**——Opus 4.6 auto-compaction 减缓此需求 |
| Git 锁文件 | 模型不会互相沟通 | **仍成立**——多进程仍需互斥 |
| `current_tasks/*.txt` 锁 | 没有中央调度器 | **仍成立**——去中心化是架构选择 |
| 16 个并发 | 单 agent 吞吐不够 | **部分失效**——Opus 4.6 单 agent 能扛更大任务 |
| Deduplicator 角色 | 多 agent 会重复造轮子 | **在多 agent 架构下仍成立** |

**推论**：Carlini 的架构有一部分是**任务特性决定的**（安全、多 agent、去中心化），有一部分是**模型能力决定的**（冷启动、细粒度并发）。前者保留，后者可随模型升级调整。

---

## 🧩 第五层感悟：Subagent 军团该学谁

对 agent-army 而言，两条路线各有应用场景：

### 适合 Carlini 路线的任务
- **有外部 oracle** 的任务（代码重构有旧版做对照、翻译有机器翻译 baseline、数据迁移有行对行校验）
- **高安全/高预算**，可以承担 Docker 隔离 + 多 agent 并发的开销
- **天然可分区**的工作（每个 agent 改一个模块、一个文件、一个微服务）
- **长周期**（数周而非数小时）

### 适合 Anthropic 路线的任务
- **无 oracle** 的主观或开放任务（做新产品、写文案、做设计）
- **需要产品级完整度**（全栈 app、端到端流程）
- **单次运行几小时**，不需要持续并发
- **依赖 Claude Agent SDK 的高级能力**（工具调用、compaction）

### 混合路线（可能是 agent-army 的甜点）

```
外层：Anthropic 式三 agent（Planner / Generator / Evaluator）处理主流程
  ↓
遇到"可并行 + 有 oracle"的子任务：
  ↓
切换到 Carlini 式 Shared State 多 agent 模式
  ↓
子任务完成，回到主流程
```

具体场景举例：
- 主流程：从 1 句话做出一个产品（Anthropic 路线）
- 子任务：产品需要迁移一个大 codebase 到新框架（Carlini 路线，有旧代码做 oracle）

---

## 🪞 对 agent-army 的直接启示

### 1. 路线选择是第一个架构决策，不是实现细节
在动手前先问：
- 任务有 oracle 吗？
- 任务需要多 agent 并发吗？
- 当前模型能 one-shot 到什么程度？

**答案决定选 Carlini 还是 Anthropic 路线。选错会事倍功半。**

### 2. 工件规范要提前设计，哪怕暂时不用
无论选哪条路线，**工件都是记忆的最终载体**：
- Carlini 路线：progress 文件、锁文件、日志格式
- Anthropic 路线：spec、bug report、sprint contract（如果用）

**一开始就定好格式**——JSON 而非 Markdown、`[ERROR] <file>:<line>` 的日志约定、强措辞的保护条款。

### 3. 独立 Evaluator 永远值得
即使在 Carlini 路线，也应该有至少一个 critic / red-team agent。
**"generator 自评会宽容"** 在两条路线下都成立——这是 LLM 的通病，不是架构选择能解决的。

### 4. 定期做 Prune Review
不管选哪条路线，**模型升级后的标准动作是 prune**：
- 每个组件的 assumption 是什么？
- 这个 assumption 在新模型上还成立吗？
- 如果不成立，组件是否应该删？

### 5. 不要试图造一套通吃的 harness
后作明确说：**Harness 空间不会收缩，它在移动**。
- Agent-army 不要做"一个永恒的 harness 基础设施"
- 做"一组可装配的 harness 模式 + 一套评估选型的方法论"

---

## 📌 一句话合观

> **Carlini 的 C Compiler 证明了"记忆全外化 + 工业化并发"在有 oracle 的任务上能走多远（100,000 行代码、可跑 Doom）。Anthropic 的 Planner-Generator-Evaluator 证明了"三角色分工 + 浏览器真测"在无 oracle 的开放任务上能走多远（1 句话到全栈 DAW）。两者不是竞争关系，是应对不同任务特性的两种正确答案——而选哪种，本身就是 harness 工程师的第一个战略决策。**

---

## 🔗 相关文档

- [`refs/parallel-claude-c-compiler/`](../refs/parallel-claude-c-compiler/) — CCC 项目细节
- [`refs/long-running-harness/`](../refs/long-running-harness/) — Anthropic 官方最终架构
- [`refs/multi-agent-coordination-patterns/`](../refs/multi-agent-coordination-patterns/) — 五模式谱系
- [`findings/ccc-multi-agent-pattern.md`](./ccc-multi-agent-pattern.md) — CCC 在五模式中的归类
- [`findings/anthropic-harness-multi-agent-pattern.md`](./anthropic-harness-multi-agent-pattern.md) — 官方 harness 在五模式中的归类
