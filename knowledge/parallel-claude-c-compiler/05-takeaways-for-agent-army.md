# 05 · 对 agent-army 项目的具体借鉴点

本文档把博客里的通用经验，翻译成 **agent-army 项目下一步可直接采用的设计决策**。

## 5.1 设计哲学上的对齐

| Carlini 的原则 | agent-army 该怎么做 |
|---|---|
| Harness 极简，智能下放到 agent | 不要造复杂 orchestrator，给 subagent 好的 prompt + 好的工具 |
| 测试是 agent 的目标函数 | 每个 skill / subagent 配套可执行 verifier |
| 为 agent 写，不是为人写 | 日志、README、错误格式全部 agent-friendly |
| 并行需要可切分 | 有无法天然切分的任务时，用 oracle 做 bisection |
| 异质化角色胜过同质化数量 | 组建 critic / dedup / docs / tester 等差异化 subagent |

## 5.2 可直接复用的技术模式

### 5.2.1 Ralph-loop（单 agent 持续推进）

适用场景：长周期、自主推进的任务（例如 "持续为某个 repo 维护 skills 目录"）。

```bash
while true; do
    opencode run \
      --subagent <agent-name> \
      --prompt "$(cat PROMPT.md)" \
      --workdir /workspace \
      >> "logs/agent_$(date +%s).log" 2>&1
done
```

### 5.2.2 Git-based 任务锁

- 使用 `tasks/in-progress/<task-id>.md` 作为锁文件
- 锁文件内容 = 任务描述 + agent identity + 时间戳
- 靠 git push conflict 实现互斥
- 完成后 move 到 `tasks/done/`

### 5.2.3 Agent-friendly 日志规范

agent-army 内部约定（建议写入 CONTRIBUTING）：

```
[ERROR] <file>:<line>: <single-line reason>
[WARN]  ...
[INFO]  ...
[SUMMARY] total=123 passed=100 failed=23 skipped=0
```

- 错误必须**单行**（grep 友好）
- 大块内容**写文件**，stdout 给路径
- 每次运行结尾输出 `[SUMMARY]` 行，字段稳定

### 5.2.4 `--fast` 采样模式

任何耗时 > 1 分钟的检查脚本，都必须支持 `--fast`：

- 默认 10% 随机采样
- seed 由 agent ID 决定（同一 agent 的采样稳定）
- 跨 agent 随机（多 agent 合起来覆盖全集）

### 5.2.5 Oracle Bisection（应对单体任务）

当 agent-army 接到 "重构整个项目"这种单体任务：

1. 把未改动的原项目作为 oracle
2. Agent 每次**只改一部分文件**，其它文件用原版
3. 自动 bisect 定位"改哪些文件会让测试挂"
4. 多个 agent 并行处理不同文件集

## 5.3 Subagent 军团的角色设计建议

基于博客里的角色分工，我们至少应有以下 subagent 原型：

### Core roles

- **`implementer`** — 写主业务代码
- **`tester`** — 写 / 维护测试、verifier、CI 配置
- **`critic`** — 不做修改，只输出"我反对，因为..."类型报告
- **`refactorer-dedup`** — 扫描重复代码并合并
- **`documentarian`** — 维护 README、progress files

### Support roles

- **`harness-maintainer`** — 维护 agent-army 自身的 infra
- **`budget-guard`** — 监控 token 消耗、发现 runaway agent

### 关键：**它们通过 git + 日志异步协作，不需要直接通信**

## 5.4 风险缓解 Checklist

- [ ] 所有 agent 在 Docker 容器内运行
- [ ] 每个 session 有 per-agent budget（token 或 $）
- [ ] Kill switch：N 分钟无 commit 自动终止
- [ ] 全量日志归档 + commit-agent 映射表
- [ ] 至少一个 critic agent 持续运行
- [ ] 高风险路径（security、public API、data migration）必须人工 gate
- [ ] CI 强制：新 commit 不能破坏已有测试

## 5.5 我们还不知道的（博客没回答的）

- **Agent 间直接通信是否值得引入？** Carlini 故意没做，但可能限制了协作深度。
- **Orchestrator agent 有用吗？** 博客说不用，但更复杂任务可能需要。
- **如何处理 agent 间知识差异？** 每个 session 是白板起步，重复读 README 代价高。
- **Skill 和 subagent 的边界如何划？** 博客没谈这个，这是 opencode 特有的抽象。

这些问题需要 agent-army 自己在实践中回答。**建议：每次尝试新模式，都像 Carlini 一样，写 benchmark、留日志、总结经验**。

## 5.6 推荐进一步阅读

- [源码仓库](https://github.com/anthropics/claudes-c-compiler) — 看 git history，观察 16 agents 真实协作记录
- 关键字搜索：`Ralph-loop`、`agent scaffolding`、`LLM test-driven development`
- 关注 Carlini 的 followup（博客说会继续更新）
