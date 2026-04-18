# 02 · Harness 架构

博客揭示的是一套**极简但有效**的 harness 架构。理解它的每一个组件比复制它更重要。

## 2.1 单 agent 的 "Ralph-loop"

Claude Code 默认是"交互式"的——完成部分任务后会等待用户输入。为了让它**持续自主推进**，Carlini 用一个 shell 无限循环把它包裹起来：

```bash
#!/bin/bash

while true; do
    COMMIT=$(git rev-parse --short=6 HEAD)
    LOGFILE="agent_logs/agent_${COMMIT}.log"

    claude --dangerously-skip-permissions \
           -p "$(cat AGENT_PROMPT.md)" \
           --model claude-opus-X-Y &> "$LOGFILE"
done
```

### 要点

- `--dangerously-skip-permissions`：避免权限提示阻塞；**必须跑在容器里**
- `-p "$(cat AGENT_PROMPT.md)"`：每次 session 用同一个 prompt 冷启动
- `&> "$LOGFILE"`：每次 session 日志按 commit hash 命名，便于归因
- `while true`：session 结束立即起新 session，**不给 Claude "完成了，停下来"的机会**

### AGENT_PROMPT.md 的核心指令（从博客描述推断）

1. 告诉它要解决什么问题
2. 让它**把问题拆成小块**
3. 让它**持续追踪**当前在做什么
4. 让它**自己决定下一步做什么**
5. 要求它**"keep going until perfect"**

> 轶事：有一次 Claude 不小心 `pkill -9 bash`，把自己连带 loop 一起杀了。说明容器隔离真的很必要。

## 2.2 多 agent 并行协作

### 仓库布局

- 一个中央 **bare git repo**（upstream）
- 每个 agent 独立的 **Docker container**
- 容器内 `/upstream` 挂载 bare repo，`/workspace` 是本地 clone
- agent 完成工作后 **push 回 upstream**

### git-based 任务锁

**没有中心调度器，用 git 的合并语义实现同步。**

1. **取锁**：Claude 写一个文本文件到 `current_tasks/`，例如：
   - Agent A: `current_tasks/parse_if_statement.txt`
   - Agent B: `current_tasks/codegen_function_definition.txt`

2. **冲突解决**：如果两个 agent 试图锁同一个任务，git push 时会产生冲突，后者被迫选另一个任务。

3. **提交流程**：
   - 做任务
   - `git pull` 同步其他 agent 的更改
   - 合并冲突（Claude 自己解决——博客说它"smart enough to figure that out"）
   - `git push`
   - 删除锁文件

4. **循环**：无限 agent-generation-loop 在全新容器里起新 session，循环往复。

### 极简的设计哲学

Carlini **刻意**没做的事：

- ❌ 没有 agent 间直接通信机制
- ❌ 没有高层目标管理进程
- ❌ 没有 orchestration agent（没有"老板 agent"）

**每个 agent 自己决定做什么**。典型行为：
- 大多数情况下，挑"下一个最明显的问题"
- 卡在某个 bug 上时，维护一份**运行中的文档**，记录"试过的失败方法"和"剩余任务"

博客里提到可以在 [git repo](https://github.com/anthropics/claudes-c-compiler) 的 history 里看到各个 agent 取锁/放锁的真实记录。

## 2.3 架构图（文字版）

```
                  ┌─────────────────────────────┐
                  │   bare git repo (upstream)  │
                  │                             │
                  │   current_tasks/*.txt  <──── 锁文件
                  │   src/, tests/, docs/       │
                  └──────────┬──────────────────┘
                             │ push/pull (冲突即信号)
         ┌───────────────────┼───────────────────┐
         │                   │                   │
  ┌──────▼──────┐     ┌──────▼──────┐    ┌──────▼──────┐
  │ Docker #1   │     │ Docker #2   │    │ Docker #16  │
  │ ┌─────────┐ │     │ ┌─────────┐ │    │ ┌─────────┐ │
  │ │Ralph    │ │     │ │Ralph    │ │    │ │Ralph    │ │
  │ │ loop    │ │ ... │ │ loop    │ │ .. │ │ loop    │ │
  │ └──┬──────┘ │     │ └──┬──────┘ │    │ └──┬──────┘ │
  │    │claude  │     │    │claude  │    │    │claude  │
  │    ▼session │     │    ▼session │    │    ▼session │
  │ /workspace  │     │ /workspace  │    │ /workspace  │
  └─────────────┘     └─────────────┘    └─────────────┘
```

## 2.4 为什么这么简单的设计能工作？

因为真正"聪明"的部分被**下放给了 Claude 本身**：
- Claude 读 README 自己理解项目状态
- Claude 自己决定做什么
- Claude 自己解决 merge conflict
- Claude 自己维护进度文档

Harness 只负责：**环境隔离、持久化、任务互斥**。

这和传统的多进程编程模型形成鲜明对比——**我们不再需要中心调度器，因为 agent 本身就有规划能力**。
