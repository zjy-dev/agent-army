# 03 · Harness 组件的耐久性与 Prune 方法论

> 这是后作最底层、最可迁移的方法论，值得单独拿一篇讲清楚。

---

## 3.1 核心命题

> **"Every component in a harness encodes an assumption about what the model can't do on its own, and those assumptions are worth stress testing, both because they may be incorrect, and because they can quickly go stale as models improve."**

每个 harness 组件都在说："**模型单独做不到 X，所以我们加一个组件补上。**"

这句话的另一面是：**模型升级后，X 可能已经做得到——这时组件变成无谓开销（overhead）甚至拖累**。

---

## 3.2 方法论原则

引自 [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents)：

> **"Find the simplest solution possible, and only increase complexity when needed."**

这不是"一上来就搭最简"，而是**持续适配**——模型动了，harness 就要重新审视。

---

## 3.3 具体案例：Sprint 的兴衰

### 诞生原因（Opus 4.5 时代）
Sprint 编码了三个假设：
1. 模型一次扛不动大任务 → 切块
2. 模型在长任务里失去连贯性 → 每块要收尾
3. 模型需要明确"下一步做什么"指令 → Sprint Contract 写死

这些假设在 **Sonnet 4.5 / Opus 4.5** 上基本成立。

### 衰亡原因（Opus 4.6）
Opus 4.6 的自带改进：
- plans more carefully
- **sustains agentic tasks for longer**
- operates more reliably in larger codebases
- **better code review and debugging skills to catch its own mistakes**
- 长 context retrieval 显著提升

**→ Sprint 承重的三个假设全部被模型能力接管。**

### 作者的具体动作
1. 拆掉 Sprint 切分，让 Generator 一气跑完
2. Evaluator 从"每 sprint gate"改为"run 结束一次"
3. 保留 Planner（没它 Generator 会 under-scope）
4. 保留 Evaluator 但角色变轻（从连续 gatekeeper 变成事后 reviewer）

### 数据验证
DAW 实验 Build Round 1 **连续跑 2 小时 7 分钟不出轨**——这是"Sprint 可以砍"的直接经验证据。

---

## 3.4 另一个案例：Context Reset 的兴衰

### 诞生原因（Sonnet 4.5 时代）
Sonnet 4.5 有 **"Context Anxiety"**：
> Context 还没满，模型就**以为**快满了 → 自保式收尾。

Compaction 救不了，因为 compaction 保留连续性但保留不了"以为快满"的焦虑。

**Context Reset** 方案：
- 清空整个 context window
- 起新 agent
- 靠结构化 handoff artifact 续上工作

代价：orchestration 复杂度、token 开销、每轮 latency。

### 衰亡原因（Opus 4.5+ 时代）
> "Opus 4.5 largely removed that behavior on its own, so I was able to drop context resets from this harness entirely."

Opus 4.5 起 context anxiety 基本消失，配合 Claude Agent SDK auto-compaction 即可维持长 session。Reset 从"必需"变"无必要开销"。

---

## 3.5 Prune 的正确方法（作者的教训）

作者尝试简化 harness 时，**第一次用了错误方法**：

### ❌ 错误做法：一把梭
> "In my first attempt to simplify, I cut the harness back radically and tried a few creative new ideas, but I wasn't able to replicate the performance of the original. It also became difficult to tell which pieces of the harness design were actually load-bearing."

**后果**：性能复现不出来，且**说不清楚哪些组件真的承重**。

### ✅ 正确做法：methodical
> "I moved to a more methodical approach, **removing one component at a time** and reviewing what impact it had on the final result."

**一次只砍一个，观察影响，然后决定是否继续砍。**

### 这是工程的常识，但在 agent harness 里格外重要
原因：
- Harness 的组件之间**相互影响难预测**——不像传统代码有清晰接口
- 模型输出的随机性让**单次实验结果不可靠**——需要重复观察
- 不同任务的 overhead 敏感度不同——某个组件在任务 A 是 overhead，在任务 B 仍承重

---

## 3.6 Prune 的触发信号

什么时候该启动一轮 prune？

| 信号 | 动作 |
|---|---|
| **新模型发布** | 标准动作：读新模型的发布说明，逐条对照 harness 组件，问"这条能力让哪个组件变 stale" |
| **成本曲线异常** | 某个组件的 token / latency 占比突然变大 —— 可能说明模型已经自己会做、组件是重复工作 |
| **实际产出退化** | 组件的存在反而降低质量（过度拟合到某种老模式）—— 尝试移除 |
| **调 prompt 改动变多** | 组件的 prompt 总得加"别这样做" —— 说明这个组件在和模型自主能力打架 |

---

## 3.7 Prune 的反义：什么时候该加

作者明确：**Prune 不是单向收缩**。

> "When a new model lands, it is generally good practice to re-examine a harness, **stripping away pieces that are no longer load-bearing to performance** AND **adding new pieces to achieve greater capability that may not have been possible before**."

模型更强 → 能跑更复杂任务 → 可能需要**以前跑不动、现在跑得动**的新组件。

### 典型情境
- 模型能用更多工具了 → 加新工具 MCP
- 模型能处理更长 context 了 → 加更丰富的 spec / 知识库
- 模型 QA 能力更强了 → 加更深度的 Evaluator 检查项
- 模型能跑更多轮迭代了 → 加更复杂的反馈循环

---

## 3.8 最终心法

> **"The space of interesting harness combinations doesn't shrink as models improve. Instead, it moves, and the interesting work for AI engineers is to keep finding the next novel combination."**

Harness 设计是一个**追踪模型能力前沿**的工程实践：

- 前沿在哪里，有意思的 harness 就在哪里
- 旧 harness 不是错，是**在旧前沿的正确解**
- 新 harness 的任务是**探索在新前沿才可能的新组合**

---

## 3.9 对 agent-army 的操作含义

1. **每个组件加一个 "assumption statement"**
   - 写清楚"这个组件假设模型做不到 X"
   - 模型升级时，逐条 review assumption

2. **定期做 "strip test"**
   - 新模型发布后跑一轮：每次移除一个组件，观察影响
   - 记录结果到 `experiments/` 或类似位置

3. **不要害怕把老组件下线**
   - 它在某个模型时代是对的，不意味着永远对
   - 保留历史在 git 里，随时可以调回来

4. **同步做 "capability test"**
   - 新模型可能解锁了以前不敢尝试的任务形态
   - 主动尝试 —— 找到只有新模型才能做的 harness
