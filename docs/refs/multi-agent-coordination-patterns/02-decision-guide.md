# 02 · 决策树与速查表

## 2.1 两两对比

### Orchestrator-Subagent vs Agent Teams

**问题：Worker 需要跨调用保持状态吗？**

- **否**（每次调用自足）→ Orchestrator-Subagent
- **是**（需要积累领域专精）→ Agent Teams

code review 系统 → Orchestrator-Subagent（每项检查跑完即返回）
codebase 迁移 → Agent Teams（每个 teammate 长期掌握一个 service）

---

### Orchestrator-Subagent vs Message Bus

**问题：工作流步骤是预先已知的吗？**

- **是**（固定 pipeline）→ Orchestrator-Subagent
- **否**（随事件涌现，新类型会出现）→ Message Bus

code review → Orchestrator-Subagent（固定流水线：接 PR → 跑检查 → 综合）
安全运营 → Message Bus（无法预知告警类型，新类型会出现）

**迁移信号**：orchestrator 里堆积越来越多 `if/else` 条件分支时，该迁移了。

---

### Agent Teams vs Shared State

**问题：Agent 需要彼此的中间发现吗？**

- **否**（干净分区，最后汇总）→ Agent Teams
- **是**（实时互相启发）→ Shared State

codebase 迁移 → Agent Teams（各 service 独立）
研究综合 → Shared State（学术发现立刻指导行业调查）

**核心判据**：一旦 teammate 之间**不只是共享最终结果**，而是**需要看见彼此的中间发现**——Shared State 是更自然的选择。

---

### Message Bus vs Shared State

**问题：工作以离散事件流动，还是累积成知识库？**

- **事件流** → Message Bus
- **累积知识** → Shared State
- 若 message bus 里的事件主要用来**分享发现**而非**触发动作**，真正想要的是 Shared State
- **需要彻底消除单点故障** → Shared State（message bus 还有 router）

---

## 2.2 速查表（原文）

| 情境 | 推荐模式 |
|---|---|
| 质量关键 + 评估标准明确 | Generator-Verifier |
| 任务分解清晰 + 子任务边界清楚 | Orchestrator-Subagent |
| 并行负载 + 独立长时任务 | Agent Teams |
| 事件驱动 pipeline + 生态会增长 | Message Bus |
| 协作研究 + agents 共享发现 | Shared State |
| 不允许单点故障 | Shared State |

---

## 2.3 起步建议

> **For most use cases, we recommend starting with orchestrator-subagent.**

理由：
- 处理的问题范围最广
- 协调开销最小
- 观察它在哪里挣扎，再往针对性方向演化

---

## 2.4 混合是常态

生产系统常见组合：

- **Orchestrator-Subagent（总流程） + Shared State（协作重的子任务）**
- **Message Bus（事件路由） + Agent Teams（每种事件类型背后的长期 worker）**
- **任何外层模式 + 内层嵌套 Generator-Verifier**（质量闭环可嵌套进任何 agent 内部）

模式是积木，不是互斥选项。
