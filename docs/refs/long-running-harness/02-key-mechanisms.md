# 02 · 核心机制

> 四个跨前后两篇文章、在 Opus 4.6 时代仍然成立的核心机制。

---

## 2.1 机制一：GAN 式的 Generator-Evaluator 分离

### 思想来源
Generative Adversarial Networks：一个生成、一个判别，用分工换质量。

### 为什么不能靠 Generator 自评
**Agent 被要求评自己的产出时，会自信地夸奖自己——哪怕质量明显平庸。** 主观任务（设计）尤其严重。

**关键洞察**（后作原文）：
> "The separation doesn't immediately eliminate that leniency on its own; the evaluator is still an LLM that is inclined to be generous towards LLM-generated outputs. **But tuning a standalone evaluator to be skeptical turns out to be far more tractable than making a generator critical of its own work**, and once that external feedback exists, the generator has something concrete to iterate against."

分离本身不直接消除宽容，但**把"调教挑剔"这个任务落到一个专职 agent 上，在工程上容易得多**。

### 实现要点
- Evaluator 和 Generator **跑独立的 context**
- Evaluator 必须有**独立的工具**（Playwright MCP）——不能只看 Generator 产出的日志
- Evaluator 的 prompt 可以调得比 Generator 苛刻得多
- 评分标准（criteria）同时喂给两边——Generator 知道自己会被什么标准量
- Few-shot 校准 Evaluator 让打分对齐作者偏好

---

## 2.2 机制二：可量化的主观评分

### 问题
"Is this design beautiful?" 没法一致回答。但——**"Does this follow our principles for good design?"** 可以。

### 解法：把主观判断拆成可分项的 criteria

后作 Frontend Design 实验的四维度评分表：

| 维度 | 定义 | 权重 |
|---|---|---|
| **Design quality** | 颜色/排版/布局/视觉组合出 coherent whole、distinct mood | 高 |
| **Originality** | 有 custom 决策？还是 template/library default/AI 套路（紫渐变白卡） | 高 |
| **Craft** | 技术功底：层次、间距、配色、对比度 | 低（模型默认就好） |
| **Functionality** | 可用性（独立于美学） | 低（模型默认就好） |

**权重偏向 design + originality**，逼模型承担审美风险。

### 措辞本身是 steering
> 在 criteria 里写 **"the best designs are museum quality"** 这句话，让设计收敛到某种特定美学。
>
> Prompt 和 criteria 的措辞**不是中性描述，它们塑造输出的性格**。

### 第一轮就已经有提升
即使没有 Evaluator 反馈，**单凭把 criteria 告诉 Generator**，输出就明显好过无 prompting 的 baseline。**Criteria 既是评分表，也是 steering 文档。**

### 示例收益
荷兰艺术博物馆案例：第 9 轮是抛光但普通的 dark-themed landing；**第 10 轮整个推翻**——做成 3D 空间体验（CSS 透视的棋盘地板、墙上挂画、用门做 gallery 间的导航）。这是单次生成拿不出的创意跳跃。

---

## 2.3 机制三：浏览器自动化 = 真实测试

### 问题
Agent 改代码 + unit test + `curl` ≠ 验证过。Agent 会在**测试覆盖不到的地方藏 bug**，甚至在复杂交互上出现"UI 看起来对但实际不响应输入"的严重问题。

### 解法：Evaluator 必须装 Playwright MCP

- 启动真实 dev server
- 开真实 browser 导航 app
- **按用户视角**逐条验证 criteria
- 截图、观察 DOM、测 API endpoint、查数据库状态
- 每条测试的结果都是**"FAIL 或 PASS + 精确证据"**

### 反面例子（文章里的 retro game maker）
Solo 版（无 harness）——UI 看起来像样，但 **entities 出现在屏幕上但不响应输入**，runtime wiring 断了。**静态代码审查 + unit test 抓不到这种问题**。

### 已知局限
| 局限 | 影响 |
|---|---|
| Claude 看不到 browser-native alert modal | 依赖这些的特性更容易 buggy |
| Claude 不能"听声音" | 音频类产品的 QA 对音乐品味判断无效（DAW 实验的局限） |

---

## 2.4 机制四：工件化的 agent 间交接

### 原则
**所有 agent 通信走文件，不走内存 / 直接通道。**

### 为什么
- 文件天然可归档、可追溯、可进 git
- 新 session 启动可以直接读文件 bootstrap
- 不依赖 "persistent memory" 的假设
- Debug 时可以人类读、人类介入

### 最小工件集（从前作继承的做法，仍然有效）

| 工件 | 谁维护 | 作用 |
|---|---|---|
| **Spec 文档**（Markdown / JSON） | Planner 建，Generator 读 | 项目全景、feature 清单 |
| **Progress 文件** | Generator 每 session 更新 | 已做了什么、下一步做什么 |
| **Git history** | Generator 每次 commit | 可回滚的增量历史 |
| **init.sh** | Generator 首次建立 | 一键启动开发环境 |
| **Bug report 文件** | Evaluator 写，Generator 读 | 本轮失败的具体证据 |

### Feature list 用 JSON 不用 Markdown（前作的经验，仍然适用）
原文理由：
> "the model is **less likely to inappropriately change or overwrite JSON files** compared to Markdown files"

加上强措辞：**"It is unacceptable to remove or edit tests"**——防止 Generator 自己改测试让自己过（Goodhart's law 的对策）。

### 文件通信的 Agent 对话样例
```
Planner  → writes  spec.md
Generator → reads  spec.md
Generator → writes code + progress.md + git commits
Generator → runs  init.sh && smoke test
Generator → writes  self-eval.md
Evaluator → reads  self-eval.md + runs Playwright
Evaluator → writes  bug-report.md
Generator → reads  bug-report.md, fixes, commits, re-loops
```

**无任何 in-memory 状态**——任意一个 agent 挂了，新 agent 读文件即可续上。

---

## 2.5 历史脚手架（仅为理解前作而列，不作为推荐）

以下是 Sonnet 4.5 / Opus 4.5 时代用过但**在 Opus 4.6+ 已被废弃**的机制。此处列出是为了让你在读前作时不被混淆。

| 旧机制 | 解决的问题 | 为什么被废弃 |
|---|---|---|
| **Context Reset** | Sonnet 4.5 的 "context anxiety"（以为快到上限就自保式收尾） | Opus 4.6 没有 context anxiety，SDK auto-compaction 足够 |
| **Sprint 切分** | Opus 4.5 长任务失去连贯性 | Opus 4.6 能连续 2+ 小时不出轨 |
| **Sprint Contract 谈判** | 把高层 spec 翻译成可测标准、按 sprint 交接 | 没有 sprint 就没有 contract；改为 run-level Evaluator 反馈 |
| **Initializer Agent**（前作的双 agent 中的第一个） | 统一搭建环境工件 | 职责被吸收进 Planner（产品 spec）和 Generator（代码/环境） |

**一句话**：这些机制都是**"模型无能的假设的编码"**，模型进步让假设失效，组件随之被 prune。这个原则见 [`03-stale-assumptions.md`](./03-stale-assumptions.md)。
