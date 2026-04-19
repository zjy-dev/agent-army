# 01 · 最终三 Agent 架构

> 基于 Opus 4.6 的实证结果。Sprint、Context Reset 等早期脚手架已被移除。

---

## 1.1 架构总览

```
User prompt (1-4 句)
       ↓
   ┌───────────┐
   │  Planner  │  扩展为完整 product spec
   └─────┬─────┘
         ↓ (文件交接)
   ┌───────────┐
   │ Generator │  连续 session 实现应用（单次可达 2+ 小时）
   └─────┬─────┘
         ↓ (运行中的 app)
   ┌───────────┐
   │ Evaluator │  Playwright 实际点击，产出 bug list + 评分
   └─────┬─────┘
         ↓ 有问题？
      ┌──┴──┐
     Yes   No
      ↓     ↓
  回到 Gen  完成
```

**外层循环**：Generator ↔ Evaluator 的多轮 review。DAW 实验跑了 3 轮 Build + QA。

---

## 1.2 Planner Agent

### 职责
- 输入：**1-4 句**用户 prompt
- 输出：**完整的 product spec**（feature 清单、用户故事、数据模型、必要的视觉语言）

### 关键 prompt 策略
| 策略 | 理由 |
|---|---|
| **Ambitious about scope** | 让产品有产品级深度，不是 demo |
| **只定产品和高层技术方向** | 不定细节——如果 planner 把技术细节写错，错误会向下游级联；让 agent 在执行中决定路径更稳 |
| **显式要求编织 AI features** | 让产出带有能通过 prompt 驱动的 agent 能力 |
| **给 planner 访问 frontend design skill** | Planner 会据此生成视觉设计语言（颜色、排版、整体感觉） |

### 示例（文章 Appendix）
> Prompt: `Create a 2D retro game maker with features including a level editor, sprite editor, entity behaviors, and a playable test mode.`
>
> Planner 扩出：16 个 feature，包括 Project Dashboard、Sprite Editor、Level Editor、Entity Behavior System、Playable Test Mode、Sprite 动画系统、行为模板、音效与音乐、**AI 辅助 sprite 生成器和 level designer**、游戏导出和分享链接。

### 为什么 Planner 必须保留
- 没有 Planner：Generator 面对 raw prompt 会 **under-scope**，直接开干不先 spec，产出功能稀少
- Planner ≠ Initializer agent（前作）——Planner 不再负责建 init.sh、git repo 等基础环境，这些由 Generator 在工作中建立

---

## 1.3 Generator Agent

### 职责
- 读 Planner 产出的 spec
- **一次 session 连续实现整个 app**（Opus 4.6 可达 2+ 小时不出轨）
- 使用 git 做版本管理，自己提交 commit

### 技术栈（文章实验用）
- React + Vite + FastAPI + SQLite / PostgreSQL
- Claude Agent SDK 提供 auto-compaction
- 可用 git 做自救（回退坏 commit）

### 关键 prompt 策略
| 策略 | 理由 |
|---|---|
| **让 Generator 自己 plan + 自己拆分** | Opus 4.6 能 natively 做到，不需要外部 Sprint 切分 |
| **在结束前做自我 smoke test** | 抓明显 bug，为 Evaluator 节省时间 |
| **构建真正的 AI agent 功能**（非只调 API） | 要求把产品内的 AI 特性做成带 tool use 的 agent |

### 实际表现（DAW 实验）
| Phase | 时长 | 成本 |
|---|---|---|
| Build Round 1 | 2h 7min | $71.08 |
| Build Round 2 | 1h 2min | $36.89 |
| Build Round 3 | 10.9min | $5.88 |

**Round 1 连续 2 小时没出轨**——这是 Opus 4.6 能"砍掉 Sprint"的直接证据。

---

## 1.4 Evaluator Agent

### 职责
- 装 **Playwright MCP**，像真实用户一样点击运行中的 app
- 测 UI、API endpoint、数据库状态
- 按评分标准打分 + 写详细 bug 报告

### 评分标准（沿用 frontend design 实验，扩展到全栈）
| 维度 | 含义 |
|---|---|
| **Product depth** | 功能完整度是否达到 spec 要求 |
| **Functionality** | 实际能不能用 |
| **Visual design** | 视觉质量（继承自 frontend design skill） |
| **Code quality** | 代码工程质量 |

**每个维度有硬阈值，任一低于阈值 → 本轮失败 + 详细反馈回 Generator**。

### Evaluator 的典型输出格式（实际例）
```
Contract criterion: Rectangle fill tool allows click-drag to fill a rectangular area
Finding: FAIL — Tool only places tiles at drag start/end points instead of filling.
        fillRectangle function exists but isn't triggered properly on mouseUp.
```

这种**精确到文件/行号/条件表达式**的反馈，是 Generator 修 bug 不需要额外调查的前提。

### 调教 Evaluator 的工程

Claude 开箱即用时是个**糟糕的 QA**：
- 认出真问题后**说服自己"这不要紧"**，批准放行
- 倾向表面测试，不探 edge case

**调优循环**：
1. 读 Evaluator 日志
2. 找"它的判断和我的判断分歧"的样本
3. 更新 QA prompt 针对这些分歧
4. 重复，直到 Evaluator 的判断可接受

**经验**：Evaluator 比 Generator 更需要精细 prompt 工程。

---

## 1.5 Agent 间通信

**全部用文件**：

- Planner 写 spec 文件 → Generator 读
- Generator 写代码和自评 → Evaluator 读
- Evaluator 写 bug list / 评分 → Generator 读并修

**不使用**直接对话通道。这让 artifact 天然可归档、可追溯、可在 git 里留痕。

---

## 1.6 Evaluator 的价值曲线（最关键的洞察）

> **"The evaluator is not a fixed yes-or-no decision. It is worth the cost when the task sits beyond what the current model does reliably solo."**

Evaluator 的相对价值随模型升级在变化：

| 任务 × 模型 能力比值 | Evaluator 价值 |
|---|---|
| 任务在模型能力**边界内** | 无谓开销，应砍 |
| 任务在模型能力**边界上** | 真实 lift，保留 |
| 任务在模型能力**边界外**很远 | 必需，甚至要升级到多轮深度 QA |

**含义**：Evaluator 不是永久固件，是可插拔的、**随任务而定**的组件。
