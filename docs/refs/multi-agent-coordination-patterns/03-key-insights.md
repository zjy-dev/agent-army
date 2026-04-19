# 03 · 隐藏但关键的设计洞察

文章正文之下的深层观察，按重要性排序。

---

## 1. "早期胜利陷阱"（Early Victory Problem）

Generator-Verifier 的头号杀手：没有硬标准的 verifier 会制造"我们有质量控制"的幻觉。

**推广**：所有多 agent 系统共通——**评估能力 = 系统上限**。模型再强，如果你无法把"好"编码成可检验的规则，系统就无法稳定优化。

---

## 2. Subagent 短命 vs Teammate 长命——概念分水岭

Orchestrator-Subagent 和 Agent Teams 看起来都是"一个协调者派发给多个工人"，实际分水岭是**状态寿命**：

- Subagent：完成 bounded 任务 → 终止 → 状态清零
- Teammate：**跨任务累积上下文** → 形成领域专精

**容易混淆的点**：进程是否长存不是判据，**上下文是否跨任务累积**才是。一个每次冷启动的长存进程，本质上还是 Subagent 语义。

---

## 3. Orchestrator 的阴暗面——信息瓶颈

层级结构的代价：横向信息流必须上浮到顶再下发，几跳之后信息失真。

这个观察是 Message Bus 和 Shared State 存在的根本理由——当 agent 之间的信息交换频繁到 orchestrator 成瓶颈时，就该去中心化。

---

## 4. Message Bus 的静默失败

比崩溃更可怕——路由错或丢事件，系统**不报警地摆烂**。

**运维含义**：可观测性投入比其他模式大一个数量级。事件追踪、correlation id、投递保证，都不是锦上添花而是生存必需。

---

## 5. Shared State 的 Reactive Loop 警告

文章最强烈的工程警告：

> "Duplicate work and concurrent writes have known engineering fixes. **Reactive loops are a behavioral problem and need first-class termination conditions.**"

- 重复工作 / 并发写 → 工程解法（locking / versioning / partitioning）
- Reactive loop → **行为问题，无工程解**；只能靠**前置设计的终止条件**
  - 时间预算
  - 收敛阈值（N 轮无新发现）
  - 指定"裁判 agent"判定是否足够

**典型反模式**：把 termination 当事后补丁。结果是要么无限循环、要么在某个 agent context 爆满时随意停止。

---

## 6. Context-centric Decomposition

（前作引入，本篇继承的上层原则）

> 按"每个 agent 需要什么上下文"切，而不是按"它做什么类型的工作"切。

五种模式的根本差异就是**如何管理 context 边界和信息流**：

| 模式 | Context 管理方式 |
|---|---|
| Generator-Verifier | 两个 context：生成 context + 验证 context，通过反馈消息桥接 |
| Orchestrator-Subagent | 主 context 聚焦主任务，子 context 独立探索，只回传蒸馏结果 |
| Agent Teams | 每个 teammate 维护专精 context，长期累积 |
| Message Bus | Context 按 topic 切分，agent 只看订阅的事件 |
| Shared State | **Context 外化到 store**，agent 按需读取 |

---

## 7. 混合是默认，不是例外

"production systems often combine patterns"——生产系统基本都是混合架构。

常见组合的两种形态：
- **嵌套**：外层一种模式，内层是另一种（例：Orchestrator-Subagent 外层 + 每个 subagent 内部跑 Generator-Verifier）
- **并列**：不同子系统用不同模式（例：前端接入用 Message Bus 做事件分发，后端计算用 Agent Teams 做长任务）

**设计纪律**：起步单一模式，观察瓶颈再组合，而不是一开始就上混合架构。

---

## 8. "无单点故障"把 Shared State 从选项变刚需

前四种模式都有**中心组件**：

- Generator-Verifier：verifier 挂了就停
- Orchestrator-Subagent：orchestrator 挂了就停
- Agent Teams：coordinator 挂了就停
- Message Bus：router 挂了就停

只有 Shared State 没有。**如果系统要求任何单点都不能导致全停**——直接选 Shared State，不用比较其他维度。

---

## 9. Agent Teams 对"独立性"的硬要求

文章反复强调：**独立性是 Agent Teams 的硬前提，不是可选优化**。

teammate 之间缺少"信息中继者"（不像 orchestrator 能主动路由），findings 无法流转。一旦 teammate 工作有耦合，两个后果同时发生：
1. 他们不知道彼此在做什么
2. 输出可能**冲突**（edit 同一文件 / 产生不相容的决定）

**识别信号**：如果你在 Agent Teams 架构里频繁需要"让 teammate A 知道 teammate B 的事"——架构选错了，应该迁移到 Shared State。
