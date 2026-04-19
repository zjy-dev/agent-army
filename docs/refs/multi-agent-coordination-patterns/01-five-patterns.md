# 01 · 五种模式逐一拆解

每种模式按「机制 / 适用 / 失败模式」三段式解读。

---

## 1. Generator-Verifier（生成者-校验者）

**最简单也最常见的多 agent 模式。**

### 机制

```
Generator → 产出 → Verifier → 接受 / 反馈重试
                        ↑_____________|
```

Generator 接收任务产出初版，Verifier 按标准判定：接受 or 带反馈打回。反馈路由回 Generator 产出修订版。循环直到通过，或达到最大迭代数。

### 适用

- 输出质量关键
- **评估标准能显式定义**
- 场景：客服邮件（准确性 + 品牌语气 + issue 全覆盖）、代码生成 + 测试、事实核查、rubric 评分、合规审查

### 失败模式

- **"早期胜利陷阱"**：只说 "check if output is good" 不定义什么叫 good → verifier 变橡皮图章，**制造质量控制的假象**
- 生成和验证未必可分——**评估创意和生成创意一样难**时，verifier 抓不到问题
- 迭代卡死——必须设 max iteration + fallback（升人工 / 返回最佳尝试 + 警告）

### 本质洞察

Verifier 的质量 = 系统的上限。这不是多加一个 agent，是**把评估标准从隐式变显式**的工程动作。

---

## 2. Orchestrator-Subagent（编排者-子代理）

**层级派发，子 agent 短命。**

### 机制

```
          Orchestrator
         /     |      \
    Sub-1    Sub-2    Sub-3   （各自独立 context，完成即终止）
```

Lead agent 接收任务，自行处理一部分 + 派发另一部分给子 agent。子 agent 在**独立 context**里工作，返回**蒸馏后的结论**，orchestrator 综合成最终输出。

> **Claude Code 本身就是这个模式。** 主 agent 写代码、编辑、跑命令；后台派子 agent 搜索大代码库、调查独立问题。

### 适用

- 任务分解清晰
- 子任务间依赖少
- 场景：PR 自动 review（安全 / 覆盖率 / 风格 / 架构一致性四路并行）

### 失败模式

- **Orchestrator 成为信息瓶颈**——子 A 发现的东西要传给子 B，必须经过 orchestrator；多几跳后关键细节被 summarize 丢失
- 不显式并行就串行跑——付了多 agent 的 token 成本没换到速度

### 本质洞察

Orchestrator 在管理 context 蒸馏。它让主任务上下文保持聚焦，**代价是横向信息流被阻断**。

---

## 3. Agent Teams（智能体团队）

**共享队列 + 长存 teammate + 跨任务累积上下文。**

### 机制

```
Coordinator ── shared queue
                    |
        Teammate-1 / Teammate-2 / Teammate-3   （长期活着，积累上下文）
```

Coordinator 派生多个 worker 作为独立进程。Teammate 从共享队列取任务、跨多步自主工作、发信号报完成。

### 与 Orchestrator-Subagent 的唯一关键区别

**Worker 的持久性。**

> "Teammates stay alive across many assignments, **accumulating context and domain specialization** that improve their performance over time."

Orchestrator-Subagent 的子 agent 完成一个 bounded 任务后就终止；Agent Teams 的 teammate 跨多次任务**不重置**。**这不是"进程是否长存"的区别，而是"上下文是否跨任务积累"的区别。**

### 适用

- 子任务独立
- **需要持续多步工作**
- 场景：大 codebase 框架迁移——每个 teammate 负责一个 service，长期掌握其依赖、测试、部署配置

### 失败模式

- **独立性是硬前提**——teammate 之间无法顺畅交换中间发现；如果工作互相影响，冲突悄悄发生
- 完成时间差异大——coordinator 要处理部分完成
- 共享资源（同一文件 / 数据库）→ 需要任务分区和冲突解决机制

---

## 4. Message Bus（消息总线）

**publish / subscribe + 路由解耦。**

### 机制

```
Agents ─publish→ [Router/Topics] ─subscribe→ Agents
```

Agent 订阅它关心的 topic，router 投递匹配消息。新 agent 新能力接入**不需要重写旧连接**。

### 适用

- 事件驱动 pipeline
- agent 生态会**持续增长**
- 流程不预先决定、随事件涌现
- 场景：安全运营——告警 → 分诊 → 按类型路由到不同调查 agent → 富化请求 → 响应协调

### 失败模式

- **追踪困难**——一个告警引爆 5 个 agent 的级联，调试比顺序决策难得多
- **静默失败**——路由错或丢事件，系统不崩溃但什么都没处理
- LLM-based router 提供语义灵活性，但带来自己的失败模式

### 本质洞察

这是把"条件分支"从 orchestrator 里搬到基础设施层，用**路由契约**换扩展性。

---

## 5. Shared State（共享状态）

**没有中央协调者。所有 agent 直接读写同一持久化存储。**

### 机制

```
Agent-A ↘              ↙ Agent-B
         [Shared Store]
Agent-C ↗              ↖ Agent-D
```

初始化步骤 seed 问题 / 数据集。Agent 自治运行，读 store → 行动 → 写回。终止靠：time limit / 收敛阈值 / 指定"裁判 agent"判定。

### 适用

- 协作研究、agents 需要**实时**看见彼此的发现并调整
- 场景：研究综合——学术文献 agent 发现关键研究者 → 行业报告 agent 立刻调查其公司

### 关键优势

- **无单点故障**——任何一个 agent 挂了其他照常
- Orchestrator / router 挂了系统全停；Shared State 天然鲁棒

### 失败模式

| 问题 | 性质 | 解法 |
|---|---|---|
| 重复工作 | 工程 | 任务分区 / 登记簿 |
| 并发写冲突 | 工程 | locking / versioning |
| **Reactive Loop**（token 空烧但不收敛） | **行为** | **必须一等公民的终止条件** |
| 涌现行为难预测 | 本质特性 | 接受或加硬约束 |

### 核心警告

> 重复工作和并发写有**工程解法**；**reactive loop 是行为问题**，必须有一等公民的终止条件——时间预算、收敛阈值（N 轮无新发现）、或指定"裁判 agent"。**把 termination 当事后补丁的系统要么无限循环、要么在某个 agent context 爆满时随意停止。**

### 本质洞察

去中心化换来鲁棒性和协作深度，代价是失控风险——**终止条件必须前置设计**。
