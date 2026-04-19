# 01 · 项目概览

## 任务定义

Carlini 给 Claude 的目标（高层 spec，**没有具体实现细节**）：

- 从零实现一个 **Rust 写的 C 编译器**
- **无外部依赖**（只依赖 Rust 标准库，clean-room 实现，全程无互联网访问）
- **GCC 兼容**
- 能 **编译 Linux 内核**
- 支持 **多个后端**（x86、ARM、RISC-V）
- 建议采用 **SSA IR** 以支持多 pass 优化

> 注意：Carlini **没有**告诉 Claude 怎么实现 SSA、怎么做寄存器分配、怎么做代码生成——他只给了架构级约束。

## 规模指标

| 指标 | 数值 |
|---|---|
| 并行 agent 数 | 16 |
| 开发周期 | ~2 周 |
| Claude Code sessions | ~2000 |
| 输入 token | 2,000,000,000 |
| 输出 token | 140,000,000 |
| API 总成本 | ~$20,000 |
| 最终代码行数 | 100,000 行 Rust |
| 模型 | Claude Opus 4.6 |

## 成果

**成功做到的：**

- 编译可启动的 Linux 6.9（x86、ARM、RISC-V）
- 编译 QEMU、FFmpeg、SQLite、postgres、Redis
- [GCC torture test suite](https://gcc.gnu.org/onlinedocs/gccint/Torture-Tests.html) **99% 通过率**
- **能编译并运行 Doom**（开发者的终极 litmus test）

**没做到的：**

- 没有 16-bit x86 编译器（boot real mode 用的），仍需调用 GCC
- 汇编器和链接器仍有 bug，demo 用的是 GCC 的
- 生成代码效率低：即使开启全部优化，也不如 GCC 关闭优化
- Rust 代码质量"还行"，但远不如专家级水准
- 不是真正的 drop-in GCC 替代品

## 博客的真正目的

Carlini 明确说：**这是一个能力基准测试（capability benchmark）**。

- 他用同一个 C compiler 项目跑过 Claude 4 整个系列
- Opus 4 时代：勉强能产生"可运行"的编译器
- Opus 4.5：第一次跨过"能过大型测试套件"的门槛，但不能编译真实项目
- Opus 4.6：可以编译 Linux kernel——**但这就是当前能力的天花板**

> "The resulting compiler has nearly reached the limits of Opus's abilities. I tried (hard!) to fix several of the above limitations but wasn't fully successful."

这意味着：**今天我们在 agent-army 中设计的 harness，其上限由当前模型能力决定**。但随着模型进步，同样的 harness 设计会解锁越来越复杂的任务。
