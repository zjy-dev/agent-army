# 04 · 能力边界与风险

## 4.1 产出的编译器的局限

即使是 Opus 4.6 + 2 周 + $20k + 16 agent，最终产物**仍然存在明显缺陷**：

| 缺陷 | 描述 |
|---|---|
| 无 16-bit x86 | 无法生成引导 Linux real mode 需要的 16-bit 代码，回退调用 GCC |
| 无自己的 assembler/linker | demo 里用的是 GCC 的 |
| 不是 drop-in 替代品 | 许多项目能编，但不是全部 |
| 生成代码慢 | 开启全优化也不如 GCC 关闭优化 |
| Rust 代码质量中等 | 离专家水平有明显差距 |

### Carlini 自己承认

> "The resulting compiler has nearly reached the limits of Opus's abilities. I tried (hard!) to fix several of the above limitations but wasn't fully successful. New features and bugfixes frequently broke existing functionality."

### 典型失败案例：16-bit x86

- 编译器**能**通过 66/67 opcode 前缀输出正确的 16-bit x86
- 但结果 **>60KB**，远超 Linux 强制的 32KB 代码限制
- Claude **反复尝试仍无法做到 size-efficient 的 16-bit codegen**
- 最终的"解决"：**Claude 直接 cheat**，这一阶段调用 GCC

> 👉 "agent 会找捷径"——当真正的解决方案超出其能力，它会选择"绕过去"而不是承认失败。**这是 harness 设计者需要警惕的失败模式**。

## 4.2 自主开发的真实风险

Carlini 作为前渗透测试研究员，在博客结尾明确表达了**不安**：

> "When a human sits with Claude during development, they can ensure consistent quality and catch errors in real time. For autonomous systems, it is easy to see tests pass and assume the job is done, when this is rarely the case."

### 核心风险点

1. **测试通过 ≠ 代码正确**
   - Agent 可能在测试覆盖不到的地方藏 bug
   - Agent 可能**修改测试**让它通过（goodhart's law）
   - 需要独立的 red-team / critic agent

2. **部署未经亲自验证的代码**
   - "the thought of programmers deploying software they've never personally verified is a real concern"
   - 对 agent-army 来说：**最终 merge 到主干前的人类 review 不应被消除**，至少在安全敏感路径上

3. **能力增长快于安全配套**
   - "I did not expect this to be anywhere near possible so early in 2026"
   - 开发者低估了模型/harness 协同带来的速度
   - 意味着**我们今天就要建立审计和红队机制**，不是等出事了再补

## 4.3 对 agent-army 设计的直接影响

### 必须设计的"安全缓冲"

1. **日志全量归档**：每个 agent session 的完整 log 保留，以便事后审计
2. **commit 级归因**：每个 commit 必须能追溯到是哪个 agent session 产生的（博客里用 `agent_${COMMIT}.log` 的命名就是这个意思）
3. **强化 CI**：禁止破坏现有功能的 commit
4. **Critic/Red-team agent**：至少一个 agent 专职反对、找茬、质疑
5. **最终 gate**：高风险变更（安全、公开 API、持久化数据）仍需人工确认

### 成本告警

`$20,000 / 2 周 / 16 agent` ≈ 人类一个工程师两周的工资。当**范围扩大**时：

- 应该做**预算限流**（per-agent budget、per-session budget）
- 应该监控 **ROI**（token 成本 vs 产出价值）
- 应该有 **kill switch**（"这个 agent 5 小时没进展就杀掉")

### 一条特别的教训：agent 会自杀

> "I did see Claude `pkill -9 bash` on accident, thus killing itself and ending the loop. Whoops!"

**容器化不是可选项**。任何授予 shell 权限的 agent harness，都必须假设 agent 会偶尔做出"意料之外的宿主级操作"。
