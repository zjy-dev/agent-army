# 04 · 对 agent-army 的借鉴点

把文章的五模式谱系翻译成本项目下一步可用的设计决策。

---

## 4.1 本项目与五模式的天然对齐关系

| 本项目现有元素 | 对应模式 |
|---|---|
| Ralph-loop 单 agent 循环 | 嵌套的 **Generator-Verifier**（Claude 生成 / 测试验证 / 反馈重写） |
| Carlini C Compiler 的 16 并行 agent + git 共享 | **Shared State**（见 `findings/ccc-multi-agent-pattern.md`） |
| OpenCode subagent 调用机制 | **Orchestrator-Subagent**（主 agent 派发给子 agent） |
| Ralph-orchestrator 的事件驱动 + Hat 角色 | 接近 **Message Bus**（需进一步核实） |

---

## 4.2 推荐的演化路径

遵循文章元方法论："**start with the simplest pattern that could work**"。

### Stage 1：从 Orchestrator-Subagent 起步

OpenCode 本身天然支持这个模式——主 agent + Task tool 派发子 agent。**这是大多数 agent-army 任务的默认起点**。

**适用场景**：
- 任务可清晰分解
- 子任务 bounded、短命
- 主任务需要保持主线 context

**起步 checklist**：
- [ ] 主 agent prompt 明确"何时派子 agent、何时自己做"
- [ ] 每个 subagent 类型配套 acceptance test
- [ ] 子 agent 返回**蒸馏后结论**，不是原始日志

---

### Stage 2：观察瓶颈，针对性演化

按文章的演化信号：

| 观察到的瓶颈 | 演化方向 |
|---|---|
| 主 agent 里堆积 `if/else` 条件分支 | → Message Bus |
| 子 agent 每次冷启动重复读 README 代价高 | → Agent Teams（需要真正持久的上下文，不只是长存进程） |
| 子 agent 之间频繁需要交换中间发现 | → Shared State |
| 质量不稳定、hallucination 频繁 | → 内嵌 Generator-Verifier |

---

### Stage 3：生产形态——混合架构

本项目最可能落在的形态：

```
外层：Orchestrator-Subagent（OpenCode 主 agent 派发）
  ├─ 协作重子任务：Shared State（用 git 或 docs/ 做 store）
  ├─ 每个子 agent 内部：Generator-Verifier（用 CI / 测试验证）
  └─ 长运行自主任务：Shared State（参考 CCC 模式，见 findings）
```

---

## 4.3 五模式对 subagent 军团角色设计的直接指导

`parallel-claude-c-compiler/05-takeaways-for-agent-army.md:72-86` 已规划的角色，按五模式重新定位：

| 角色 | 更适合的模式 | 理由 |
|---|---|---|
| `implementer` | Orchestrator-Subagent | 任务 bounded，完成即返回 |
| `tester` | Generator-Verifier 的 verifier 角色 | 正是 verifier 本职 |
| `critic` | Generator-Verifier 的 verifier 角色 | 天然是第二轮验证 |
| `refactorer-dedup` | Shared State | 需要看到所有其他 agent 的代码产出，findings 实时流动 |
| `documentarian` | Shared State | 需要持续观察仓库状态，写回文档 |
| `harness-maintainer` | Orchestrator-Subagent | 短命、bounded |
| `budget-guard` | Message Bus | 监听所有 session 的 token 事件，是天然的事件驱动 |

**洞察**：不同角色适合不同模式——**异质化军团本身就是混合架构**。

---

## 4.4 终止条件是一等公民（最硬的约束）

文章对 Shared State 的最强警告，对本项目的直接影响：

**任何启用 Shared State 的子系统必须在 Day 1 就设计终止条件。** 不是等跑起来看效果，而是**前置设计**。

可选机制：
- **时间预算**：每个任务最多跑 N 分钟 / 消耗 N token
- **收敛阈值**：N 轮无新 commit / N 轮无新 finding
- **裁判 agent**：专门一个 agent 判定"是否足够好了"
- **外部 oracle**：测试套件通过率达到阈值 → 终止

**反模式**：在 Ralph-loop 外层直接套 `while true` 而不设退出条件——这就是文章警告的"treat termination as an afterthought"。

---

## 4.5 还需要在实践中回答的问题

文章没回答的，本项目要自己摸索：

1. **OpenCode subagent 和 skill 在五模式中如何定位？** —— skill 更像 Generator-Verifier 的可复用模板，subagent 更像 Orchestrator-Subagent 的子节点？
2. **Ralph-loop 的 "fresh context" 和 Agent Teams 的 "accumulated context" 如何调和？** —— 或许 agent-army 的答案是：**记忆外化到 shared store，teammate 保持 fresh**（参考 CCC 的做法）
3. **如何为混合架构做可观测性？** —— Message Bus 的静默失败、Shared State 的 reactive loop，都需要专门的监控机制

这些问题的答案只能通过**实践 + 留日志 + 复盘**积累，就像 Carlini 在 C Compiler 项目里做的那样。
