# AI Agent 工程五层演进：从 Prompt 到 Graph
## 概览
AI Agent 工程领域在 2022–2026 年间逐步分化出五个层次，从低到高构成完整技术栈：
```plain text
Graph Engineering            ← 多 Agent 编排层
    └── Loop Engineering         ← 单 Agent 迭代层
        └── Harness Engineering      ← 运行时基础设施层
            └── Context Engineering      ← 信息供给层
                └── Prompt Engineering       ← 指令设计层
```
---
## 1. Prompt Engineering（提示工程）
| 项目 | 内容 |
| --- | --- |
| 核心问题 | 怎么说 |
| 作用域 | 单次 LLM 调用 |
| 时间线 | 2022–2023 |
**定义**：精心设计发给 LLM 的单次指令文本——system message、用户指令、输出格式约束等。
**核心关注**：
- 措辞、语气、角色设定
- Few-shot 示例
- Chain-of-Thought 引导
- 输出格式（JSON schema、markdown 等）
**局限**：只优化了"一次调用"的效果，无法解决动态信息注入、多轮迭代、工具调用等问题。
---
## 2. Context Engineering（上下文工程）
| 项目 | 内容 |
| --- | --- |
| 核心问题 | 模型看到什么 |
| 作用域 | 单次调用的完整输入 |
| 时间线 | 2025 |
| 推动者 | Andrej Karpathy |
**定义**：工程化管理模型在每次推理时能"看到"的全部信息——不仅是 prompt 本身，而是整个 context window 的构造。
**核心关注**：
- 动态检索（RAG）：什么时候检索、检索什么、怎么排序
- 记忆系统：短期/长期记忆的存取策略
- 上下文压缩与裁剪：token 有限时保留哪些信息
- 多源信息编排：system prompt + 用户历史 + 检索文档 + 工具结果的组装顺序
**与 Prompt Engineering 的关系**：Prompt 是写好"指令"，Context 是构建模型的完整"认知环境"。Prompt 是 Context 的子集。
---
## 3. Harness Engineering（框架工程）
| 项目 | 内容 |
| --- | --- |
| 核心问题 | 模型能做什么 |
| 作用域 | Agent 运行时环境 |
| 时间线 | 2025 下半年–2026 |
**定义**：围绕 LLM 构建的确定性运行时框架，即模型"外面那一圈"基础设施。
**核心关注**：
- 工具注册与调用管理（MCP、function calling）
- 权限与沙箱隔离
- Schema 校验（输入/输出格式强制约束）
- 执行环境（文件系统、网络、数据库访问）
- [AGENTS.md](http://agents.md/) / [CLAUDE.md](http://claude.md/) 等声明式配置
- 可观测性（日志、trace、eval）
- 错误恢复策略
**典型代表**：Claude Code 本身就是一个 harness——它定义了工具集、权限模型、memory 系统、计划模式等，模型在这个"笼子"里运转。
**与 Context Engineering 的关系**：Context Engineering 决定模型"看到什么"，Harness Engineering 决定模型"能做什么"以及"怎么安全地做"。
---
## 4. Loop Engineering（循环工程）
| 项目 | 内容 |
| --- | --- |
| 核心问题 | 怎么迭代收敛 |
| 作用域 | 单 Agent 多轮执行 |
| 时间线 | 2025 底–2026 上半年 |
**定义**：设计 Agent 的迭代反馈循环——让模型反复执行-检查-修正，直至收敛到目标状态。
**核心关注**：
- 收敛条件设计：怎么判断"够好了，可以停了"
- 自校正机制（Reflexion、self-critique）
- 重试策略与退出条件（避免无限循环）
- 验证器/判定器（linter、test suite、eval model 作为循环终止信号）
- 状态缩减：每轮循环后怎么精简上下文
**典型模式**：
```plain text
while not converged:
    plan → act → observe → evaluate → refine
```
**关键洞察**（a16z）：
> "Loop engineering 不是让 agent 重试，而是让每一轮循环都缩短当前状态与目标状态之间的距离。"
**与 Harness 的关系**：Harness 提供了工具和沙箱，Loop 在其中设计"用多少轮、怎么判停、怎么反馈"的策略。
---
## 5. Graph Engineering（图工程）
| 项目 | 内容 |
| --- | --- |
| 核心问题 | 多 Agent 怎么协作 |
| 作用域 | 多 Agent 系统 |
| 时间线 | 2026.7（Peter Steinberger 推文后爆发） |
**定义**：将 AI 工作流设计为一个节点图（DAG 或有环图），其中每个节点是一个专职 Agent，边代表数据流和控制流。
**核心关注**：
- 多 Agent 拓扑设计（串行、并行、条件分支、汇聚）
- 节点专职化：每个 Agent 只做一件事（检索、推理、验证、格式化…）
- Agent 间通信协议
- 条件路由与动态编排
- 跨节点的状态传递与一致性
**与 Loop Engineering 的关系**：
- Loop = **单 Agent** 自己检查自己的工作（纵向深度）
- Graph = **多个专职 Agent** 互相协作、互相验证（横向广度）
**典型模式**：
```plain text
[Researcher] → [Analyzer] → [Critic] → [Writer]
                    ↑                        |
                    └────── feedback ─────────┘
```
---
## 总结对比
| 维度 | Prompt Eng. | Context Eng. | Harness Eng. | Loop Eng. | Graph Eng. |
| --- | --- | --- | --- | --- | --- |
| 核心问题 | 怎么说 | 看到什么 | 能做什么 | 怎么迭代 | 怎么协作 |
| 作用域 | 单次调用 | 单次调用的输入 | Agent 运行时 | 单 Agent 多轮 | 多 Agent 系统 |
| 关键产出 | 好 prompt | 好 context window | 安全可靠的工具环境 | 收敛的反馈循环 | 高效的 Agent 拓扑 |
| 时间线 | 2022–2023 | 2025 | 2025H2 | 2025底–2026 | 2026.7 |
| 类比 | 写好问题 | 准备好考试资料 | 建好考场 | 反复检查答案 | 组建团队协作 |
---
## 备注
- 每一层都依赖下面的层：Graph 中的每个节点 Agent 在 Loop 中迭代，Loop 运行在 Harness 里，Harness 每轮调用时做 Context Engineering，Context 中的 system prompt 部分就是 Prompt Engineering。
- "Graph Engineering" 最初是 Peter Steinberger 在 2026-07-18 以半讽刺方式发出的推文（讽刺 buzzword 泛滥），但因描述了真实工程问题而被社区认真采纳。
- 社区中也有声音认为这些只是"同一件事的不同视角"——但从实践角度看，它们确实对应了构建 AI Agent 系统时不同层次的设计决策。