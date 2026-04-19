# 概念 1：仓库即记录系统

## 原文要点

智能体在运行时无法访问的任何内容，对它来说都**不存在**。

知识的存放位置决定了它是否有效：

| 位置 | 对人类 | 对智能体 |
|------|--------|----------|
| Google Docs | ✅ | ❌ |
| Slack 讨论 | ✅ | ❌ |
| 团队成员脑中 | ✅ | ❌ |
| 仓库内 Markdown | ✅ | ✅ |
| 代码 + 注释 | ✅ | ✅ |
| Lint 规则 | 间接 ✅ | ✅（强制） |

## 文档结构（原文方案）

```
AGENTS.md              ← 入口目录 (~100行)
ARCHITECTURE.md        ← 域和包分层的顶层地图
docs/
├── design-docs/       ← 设计决策，带验证状态
├── exec-plans/        ← 执行计划，带进度和决策日志
│   ├── active/
│   └── completed/
├── product-specs/     ← 产品规格
├── references/        ← 外部参考（llms.txt）
├── generated/         ← 自动生成（DB schema 等）
├── QUALITY_SCORE.md   ← 每个领域的质量评分
├── RELIABILITY.md
├── SECURITY.md
└── ...
```

## 关键实践

1. **AGENTS.md 是目录，不是百科** — ~100行，只指路
2. **专职 linter + CI 验证** — 知识库是否更新、是否交叉链接、结构是否正确
3. **doc-gardening 智能体** — 定期扫描过时文档，自动发起修复 PR
4. **执行计划是一等工件** — 提交到仓库，版本控制，带进度日志
