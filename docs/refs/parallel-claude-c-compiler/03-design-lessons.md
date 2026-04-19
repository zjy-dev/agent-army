# 03 · 四条核心设计经验

Carlini 在博客里用四个小节总结了"经验教训"。这是整篇博客对 harness 工程师**最直接的可执行指导**。

---

## Lesson 1: Write Extremely High-Quality Tests

> "Claude will work autonomously to solve whatever problem I give it. So it's important that the task verifier is nearly perfect, otherwise Claude will solve the wrong problem."

### 含义

在 autonomous 模式下，**测试就是 agent 的目标函数**。测试不完善 = agent 优化错目标。

### 做法

- 找**现成的高质量编译器测试套件**（如 GCC torture test）
- 为开源软件包写 verifier 和 build script
- **观察 Claude 的错误模式**，针对失败模式**设计新测试**
- 在项目后期，Claude 开始频繁"修新功能时打破老功能"，Carlini 的应对是：
  - 搭 **CI pipeline**
  - 加入 **更严格的 enforcement**（不允许新 commit 破坏现有代码）

### 对 agent-army 的意义

> **子 agent 的 verifier（skill 的测试/示例）不完善，军团就在"高效地朝错方向飞奔"。**

每个 subagent 定义时，**必须配套可自动化验证的 acceptance test**。

---

## Lesson 2: Put Yourself in Claude's Shoes

> "I had to constantly remind myself that I was writing this test harness for Claude and not for myself."

这是整篇博客最**反直觉**也最重要的一条。

### 设计 harness 不是给人看的

**Agent 每次启动都是白板**。它没有你开发时积累的上下文。所以：

#### 2a. 维护大量 README 和进度文件

- 要求 agent **频繁更新**当前状态
- README 要写给"一个刚进容器、什么都不知道的新 Claude"看
- **不要**假设任何"大家都知道的上下文"

#### 2b. Context Window Pollution（上下文污染）

**绝对不要**让测试 harness 打印成千上万行无用字节。

正确做法：
- Stdout 只打印**几行关键输出**
- 所有重要信息写到**文件**，需要时 Claude 自己 grep
- 日志要**机器可处理**：错误行统一带 `ERROR` 前缀，理由写在**同一行**，这样 `grep ERROR logfile` 能一次性拿到所有错误
- **预先计算聚合统计**（总数、通过率等），Claude 不用自己算

#### 2c. Time Blindness（时间盲视）

Claude **感知不到时间流逝**。放着不管，它会心甘情愿跑几小时测试而无任何进展。

应对：
- 测试 harness **只在稀疏的时间点**打印增量进度（避免污染 context）
- 提供 **`--fast` 模式**作为默认：
  - 随机跑 **1% 或 10% 采样**
  - 采样是 **per-agent 确定性 + 跨 VM 随机**
  - 所有 agent 合起来覆盖全部文件，**每个 agent 能精确定位 regression**

### 对 agent-army 的意义

> 每个 skill/subagent 的**输出格式**都应该为"下一个 agent 消费"优化，而不是为人类观察者优化。

- `grep` 友好
- 错误集中、一行一条
- 聚合统计前置
- 长输出写文件、返回路径

---

## Lesson 3: Make Parallelism Easy

> 并行性不是 "起多个进程"就完事，**工作必须可切分**。

### 两种场景

#### 场景 A：天然并行（Easy Mode）

测试套件里有 **几百个独立失败的测试** → 每个 agent 认领一个失败的 test，天然并行。

#### 场景 B：单体任务（Hard Mode）

**编译 Linux 内核就是一个巨大的单体任务**。问题：

- 每个 agent 跑编译，都撞上同一个 bug
- 都去修同一个 bug
- **相互覆盖对方的修改**
- 16 个 agent 并行 ≈ 1 个 agent 的效果

#### 解法：用 Oracle 做 Bisection 人为切片

Carlini 用 **GCC 作为 online known-good 编译器 oracle** 对比：

1. 写一个新 test harness：**随机用 GCC 编译绝大部分 kernel 文件，只留剩余文件给 Claude 的编译器**
2. 如果 kernel 最终能跑 → 问题**不在** Claude 负责的那批文件里
3. 如果 kernel 挂了 → 继续把 Claude 负责的子集**二分**，用 GCC 替换一半再试
4. 这样每个 agent 可以**并行**在不同文件里修不同的 bug
5. 最终 Claude 能独立编译全部文件时，还要用 delta debugging 找"单独都过但一起就炸"的文件对

### 对 agent-army 的意义

> 当主任务无法天然切分时，**引入一个已知正确的 oracle**（另一个工具、另一个 agent、历史正确实现）来做 bisection / fallback，就能把单体任务切成可并行单元。

例子类比：
- 代码重构：用旧代码作 oracle，逐模块替换
- 翻译：用机器翻译 baseline 作 oracle，agent 只改问题段
- 数据迁移：用旧系统为 oracle 做行对行校验

---

## Lesson 4: Multiple Agent Roles（并行 = 专业化的机会）

一旦有了并行基础设施，就不是所有 agent 都做同样的事。**不同角色的 agent 并行工作**：

| 角色 | 职责 |
|---|---|
| Builder agents | 解决主任务的 bug |
| Deduplicator | 发现并合并重复实现的代码（LLM 写代码常重复造轮子） |
| Perf: Compiler-self | 优化编译器本身的性能 |
| Perf: Generated-code | 优化编译器产出的代码的效率 |
| Rust Critic | 以 Rust 专家视角评审，做结构性重构 |
| Documentarian | 维护文档 |

### 要点

- **没有 orchestrator**，每个 agent 有自己的 prompt 和关注点
- 它们通过同一个 git repo **异步协作**
- 专业化不需要复杂的通信协议——**共享代码库 + 不同 prompt** 就够了

### 对 agent-army 的意义

> Subagent 军团的真正威力在于**异质化**。不要做 16 个相同的 coder agent，而要做：
> - 1-2 个 implementer
> - 1 个 dedup/refactor agent
> - 1 个 tester / test-harness 维护 agent
> - 1 个 docs/README 守护 agent
> - 1 个 critic / architect

他们共享的只有：**codebase + agreed-upon logging format + task lock 机制**。
