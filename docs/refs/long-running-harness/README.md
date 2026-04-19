# Long-Running Harness（Anthropic 官方系列）

本目录是 Anthropic 两篇关于"长运行 agent harness"的工程博客的**合并精炼版**，只保留 **Opus 4.6 时代的最终结论**——早期 Sonnet 4.5 / Opus 4.5 的临时脚手架（context reset、sprint 等）只在历史背景里一笔带过，不作为推荐做法。

## 原始来源

| 文章 | 作者 | 日期 | 本目录的采用策略 |
|---|---|---|---|
| [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) | Justin Young | 2025-11-26 | 作为**问题背景**保留；其中"Initializer + Coding" 双 agent 方案已被后作部分取代 |
| [Harness design for long-running application development](https://www.anthropic.com/engineering/harness-design-long-running-apps) | Prithvi Rajasekaran | 2026-03-24 | **作为最终结论的主要来源**；砍掉 sprint、砍掉 context reset 之后的形态 |

## 索引

- [`01-architecture.md`](./01-architecture.md) — 最终三 agent 架构（Planner / Generator / Evaluator）
- [`02-key-mechanisms.md`](./02-key-mechanisms.md) — 核心机制（spec 层次、GAN 式分离、工件交接、浏览器自动化测试）
- [`03-stale-assumptions.md`](./03-stale-assumptions.md) — Harness 组件的耐久性与 prune 方法论

## 一句话摘要

> **三个 agent，一个原则，一次测试都不放过。**
>
> - 三个 agent：**Planner**（1 句话 → 完整 spec）+ **Generator**（长 session 连续实现）+ **Evaluator**（Playwright 真点真测）
> - 一个原则：**Harness 每个组件都编码了一个"关于模型无能的假设"，模型升级会让假设 stale，必须定期 prune**
> - 一次测试都不放过：**独立 Evaluator 替代自我评估**；Evaluator 的价值 = 任务复杂度 相对 模型当前能力的溢出量

## 核心立场（只保留强模型时代的结论）

### ✅ 仍然成立的做法

1. **三 agent 分化**（Planner / Generator / Evaluator）是处理超出模型单体能力任务的主骨架
2. **Generator 和 Evaluator 分离**——"调一个挑剔的独立 evaluator"比"让 generator 自我批判"容易得多
3. **Evaluator 用 Playwright MCP 实际点击**真实运行的 app，而非看静态截图或日志
4. **Planner 故意模糊**——定 deliverables 而非实现路径，避免错误向下游级联
5. **评分标准的措辞本身是 steering**——"museum quality" 这类词会塑造输出性格
6. **Evaluator 不是固定 yes/no gate**——任务在模型能力边界内时是无谓开销，边界外时才给真实 lift
7. **Claude Agent SDK 的 auto-compaction** 配合 Opus 4.6+ 足够维持长 session 连贯性

### ❌ 已被淘汰的做法（Sonnet 4.5 / Opus 4.5 时代的脚手架）

1. ~~Context reset + 结构化 handoff~~ → Opus 4.6 取消了 "context anxiety"，不再需要
2. ~~Sprint 切分 + Sprint Contract~~ → Opus 4.6 能连续跑 2+ 小时不出轨，Sprint 变无谓开销
3. ~~Initializer agent 单独一跑~~ → 被 Planner 吸收并强化（1 句话即可扩到完整 spec）
4. ~~Coding agent 自己端到端验证~~ → 已证实不可靠，必须独立 Evaluator

### 🔑 元原则

> **"Find the simplest solution possible, and only increase complexity when needed."**
>
> 新模型发布后的标准动作：**移除不再承重的部分**，**加入新能力才解锁的新部分**。
>
> Harness 设计的空间不会随模型进步而收缩——它在**移动**。

## 对 agent-army 的定位

这套架构对应 [`multi-agent-coordination-patterns/`](../multi-agent-coordination-patterns/) 五模式谱系中的：

> **Orchestrator-Subagent（外层 Planner 派发 + 收集）+ 嵌套 Generator-Verifier（Generator ↔ Evaluator 的反馈循环）**

详细归类见 [`findings/anthropic-harness-multi-agent-pattern.md`](../../findings/anthropic-harness-multi-agent-pattern.md)。

与 Carlini C Compiler 的 Shared State 模式对比见 [`findings/ccc-and-harness-synthesis.md`](../../findings/ccc-and-harness-synthesis.md)。
