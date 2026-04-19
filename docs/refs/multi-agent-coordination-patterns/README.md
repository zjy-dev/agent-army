# Multi-agent Coordination Patterns

本目录是对 Anthropic 博客 [Multi-agent coordination patterns: Five approaches and when to use them](https://claude.com/blog/multi-agent-coordination-patterns) (Cara Phillips, 2026-04-10) 的深度解读与提炼。

这是姊妹篇之二——前作 [Building multi-agent systems: when and how to use them](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them) 回答"要不要用多 agent"，本篇回答"**已经决定要用，选哪种协作模式**"。

## 索引

- [`01-five-patterns.md`](./01-five-patterns.md) — 五种模式逐一拆解（机制、适用、失败模式）
- [`02-decision-guide.md`](./02-decision-guide.md) — 两两对比决策树与速查表
- [`03-key-insights.md`](./03-key-insights.md) — 隐藏但关键的设计洞察
- [`04-takeaways-for-agent-army.md`](./04-takeaways-for-agent-army.md) — 对本项目的借鉴点

## 一句话摘要

> 用最简单够用的模式起步，观察它在哪里挣扎，再演化到下一个模式。多数情况从 **Orchestrator-Subagent** 起步；生产系统通常是**混合**的。

## 五种模式一览

| 模式 | 一句话定义 | 典型适用 |
|---|---|---|
| **Generator-Verifier** | 生成-校验循环 | 质量关键 + 评估标准可显式化 |
| **Orchestrator-Subagent** | 层级派发 + 短命子 agent | 任务可清晰分解、子任务边界清楚 |
| **Agent Teams** | 共享队列 + 长存 teammate + 跨任务累积上下文 | 独立的长时多步任务 |
| **Message Bus** | publish/subscribe + router | 事件驱动 pipeline + 生态会增长 |
| **Shared State** | 所有 agent 直接读写共享存储，**无中央协调** | 协作研究、findings 实时共享、去单点 |

## 元方法论

> **"Start with the simplest pattern that could work, watch where it struggles, and evolve from there."**

- 不要按"听起来更高级"挑模式，要按**当前问题的结构**挑
- 五种模式不是互斥选项，是**积木**
- 上层原则是 **context-centric decomposition**：按"每个 agent 需要什么上下文"切，而不是按"它做什么类型的工作"切

## 核心决策问题

按以下顺序问自己：

1. 输出质量是否有**显式可编码的评估标准**？ → Generator-Verifier（可嵌套进其他模式内部）
2. 工作流步骤**预先已知**吗？ → 是选 Orchestrator-Subagent，否选 Message Bus
3. Worker 需要**跨调用保持状态**吗？ → 是选 Agent Teams，否选 Orchestrator-Subagent
4. Agent 需要**彼此的中间发现**吗？ → 是选 Shared State，否选 Agent Teams
5. 需要**消除单点故障**吗？ → 选 Shared State
