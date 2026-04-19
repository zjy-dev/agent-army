# 概念 2：机械化执行

## 核心思想

> 通过强制执行不变量，而非对实施过程进行微观管理

文档会腐烂。人会忘记。但 lint 规则和 CI 检查每次都会执行。

## 两类约束

### 架构约束（结构测试）
- 域内分层顺序：Types → Config → Repo → Service → Runtime → UI
- 依赖方向只能向前
- 横切关注点必须通过 Providers 进入
- 违反 = CI 阻塞合并

### 品味不变式（自定义 linter）
- 结构化日志（禁止 console.log 裸输出）
- Schema/类型的命名约定
- 文件大小限制
- 平台特定的可靠性要求

## 关键设计：lint 错误信息 = 修复指令

```
❌ 普通做法：
Error: File exceeds 500 lines.

✅ Harness 做法：
Error: File exceeds 500 lines.
Fix: Split into domain-specific modules following docs/ARCHITECTURE.md#splitting-guide.
Consider extracting types to <domain>/types/ and service logic to <domain>/service/.
```

错误信息中注入智能体可执行的修复路径 → 自我纠正闭环。

## 哲学

> 在中央层面强制执行边界，在本地层面允许自主权。

类似大型工程平台组织的管理模式：
- **严格的**：边界、正确性、可重复性
- **自由的**：边界内的具体实现方式
- 生成的代码不符合人类风格偏好？没关系。正确 + 可维护 + 智能体可读 = 达标。
