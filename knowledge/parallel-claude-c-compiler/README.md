# Building a C Compiler with a Team of Parallel Claudes

本目录是对 Anthropic 工程博客 [Building a C compiler with a team of parallel Claudes](https://www.anthropic.com/engineering/building-c-compiler) (Nicholas Carlini, 2026-02-05) 的深度解读与提炼。

这篇博客对 **agent-army** 项目尤其重要——它提供了一套经过实战验证的、构建 **长时间运行、多 agent 并行协作** 的 harness 设计方法论。这正是我们打造 opencode subagent 开发军团所需要的核心知识。

## 索引

- [`01-overview.md`](./01-overview.md) — 项目背景、规模与结果
- [`02-harness-architecture.md`](./02-harness-architecture.md) — Harness 架构：Ralph-loop、Docker、git 锁机制
- [`03-design-lessons.md`](./03-design-lessons.md) — 四条核心设计经验（给 agent-army 的直接启示）
- [`04-limitations-and-risks.md`](./04-limitations-and-risks.md) — 能力边界与自主开发的风险
- [`05-takeaways-for-agent-army.md`](./05-takeaways-for-agent-army.md) — 对本项目（agent-army）的具体借鉴点

## 一句话摘要

> 用 16 个并行的 Claude Opus 4.6 实例，在近 2000 个 Claude Code sessions、耗费约 $20,000 API 费用后，**从零**写出一个 100,000 行的 Rust 实现的 C 编译器，可以编译 Linux 6.9 内核（x86/ARM/RISC-V）、SQLite、Redis、FFmpeg、QEMU、postgres，并能跑 Doom。

## 对 agent-army 最关键的启示

1. **Harness 设计 > Agent 能力本身**：Carlini 大部分精力用在"让 agent 能自己理解进度"上，而不是调 prompt。
2. **测试体系是 agent 的"眼睛"**：没有完美 verifier，agent 会去解错的问题。
3. **并行化需要"可拆分的工作单元"**：当任务是单体（如编译 Linux kernel）时，要用 oracle（GCC）做 bisection 把任务人为切片。
4. **为 agent 而写，不是为人而写**：日志、文档、README 要围绕 agent 的认知限制（context pollution、time blindness）设计。
