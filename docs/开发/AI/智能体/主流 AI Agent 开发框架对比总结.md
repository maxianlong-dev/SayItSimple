# 主流 AI Agent 开发框架对比总结
> 更新时间：2026年8月
	覆盖范围：开源框架 + 大厂官方 SDK + 低代码平台
---
## 一、框架总览
| 框架 | 开发方 | GitHub Stars | 语言 | 核心模式 | 定位 |
| --- | --- | --- | --- | --- | --- |
| **LangGraph** | LangChain | \~25K | Python/JS | 图（Graph）模型 | 精细控制的状态机工作流 |
| **CrewAI** | 社区开源 | \~28K | Python | 角色（Crew）模型 | 多角色协作团队 |
| **AutoGen** | 微软 | \~38K | Python | 对话（Conversation）模型 | 多 Agent 对话协商 |
| **OpenAI Agents SDK** | OpenAI | — | Python | 轻量编排 | OpenAI 生态原生 Agent |
| **Google ADK** | Google | — | Python | 混合（图+角色） | Google Cloud 原生 Agent |
| **Semantic Kernel** | 微软 | \~22K | C#/Python/Java | 插件模型 | 企业级 AI 编排 |
| **MetaGPT** | 社区开源 | \~45K | Python | SOP 角色模型 | 模拟软件公司多角色协作 |
| **DeerFlow** | 字节跳动 | \~79K | Python | SuperAgent Harness | 深度研究 + 代码执行 |
| **DeepAgent** | Abacus AI / 学术 | — | Python | 可扩展工具集推理 | 通用推理 Agent |
| **ModelScope-Agent** | 阿里巴巴 | — | Python | 工具注册 + 多模型 | 开源模型 Agent 开发 |
| **Dify** | 社区开源 | \~50K+ | Python/TS | 低代码可视化 | 低代码 Agent 平台 |
| **Coze** | 字节跳动 | — | — | 低代码可视化 | 面向消费者的 Agent 平台 |
---
## 二、核心框架详细对比
### 1. LangGraph（LangChain 旗下）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 将 Agent 工作流建模为有向图（DAG），节点是操作，边是条件分支 |
| **核心特性** | 状态持久化、checkpoint、循环/条件分支、人机交互断点、streaming |
| **优势** | 精细控制流程；Token 效率高（比 CrewAI 快 2.2x）；已发布稳定 1.0 API |
| **劣势** | 学习曲线陡峭；简单场景 over-engineering |
| **适合场景** | 需要精确控制流程的生产级应用、复杂条件分支工作流 |
| **模型支持** | 多模型（OpenAI、Anthropic、开源模型等） |
### 2. CrewAI
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 基于角色（Role-based），定义 Agent 团队，每个 Agent 有角色、目标和背景 |
| **核心特性** | 声明式角色定义、任务委派、协作记忆、生产级 checkpoint |
| **优势** | 上手简单；社区活跃插件丰富；直觉化的角色建模 |
| **劣势** | 底层控制粒度较粗；Token 消耗相对较高；复杂流程依赖隐式调度 |
| **适合场景** | 多角色协作（研究管线、内容生产团队、商业自动化） |
| **模型支持** | 多模型 |
### 3. AutoGen（微软）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 将所有 Agent 交互建模为群聊对话（Group Chat），Agent 间通过对话协商 |
| **核心特性** | 灵活的多 Agent 对话、人机混合参与、群聊协调器、代码执行沙箱 |
| **优势** | 对话模式灵活；微软生态整合好；社区最大（38K stars） |
| **劣势** | 中央协调器瓶颈；Token 效率低（与 LangChain 有 8-9x 差距）；API 变动频繁 |
| **适合场景** | 研究探索型多 Agent 系统、需要 Agent 讨论/辩论的场景 |
| **模型支持** | 多模型（Azure OpenAI 优先） |
### 4. OpenAI Agents SDK
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 从实验性 Swarm 项目演化为生产 SDK，轻量级 Agent 编排 |
| **核心特性** | 内置 tracing、sandbox、tool use、handoff 机制、guardrails |
| **优势** | 与 OpenAI 模型深度集成；API 简洁上手快；内置安全护栏 |
| **劣势** | 强绑定 OpenAI 生态；多 Agent 编排能力相对基础 |
| **适合场景** | 已深度使用 OpenAI API 的团队、快速原型 |
| **模型支持** | OpenAI 模型（GPT-4o/o1/o3 等） |
### 5. Google ADK (Agent Development Kit)
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 混合模式，同时支持图工作流 Agent 和角色协作 Agent |
| **核心特性** | Workflow Agent + Conversational Agent、A2A 协议、Vertex AI 集成 |
| **优势** | Google Cloud 原生集成；支持 A2A 协议；混合编排灵活 |
| **劣势** | 生态绑定 Google Cloud；相对较新（2025 发布） |
| **适合场景** | Google Cloud 用户、需要混合编排模式的团队 |
| **模型支持** | Gemini 优先，支持多模型 |
### 6. Semantic Kernel（微软）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 插件化编排，以"技能（Skill）"和"规划器（Planner）"为核心 |
| **核心特性** | 多语言支持（C#/Python/Java）、插件市场、企业安全合规 |
| **优势** | 企业级成熟度高；.NET/Java 生态无缝集成；微软全家桶支持 |
| **劣势** | Python 生态不如 LangChain 丰富；社区相对封闭 |
| **适合场景** | 企业级 .NET/Java 应用、Microsoft 365 集成 |
| **模型支持** | Azure OpenAI + 多模型 |
### 7. MetaGPT
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 模拟软件公司 SOP 流程，Agent 扮演产品经理、架构师、工程师等角色 |
| **核心特性** | SOP 驱动的多角色协作、自动需求拆解、PRD/设计/代码全链路 |
| **优势** | 多角色协作标准化流程；代码生成质量较高；Stars 很高（45K） |
| **劣势** | 场景较窄（偏向软件开发）；通用性不如 LangGraph/AutoGen |
| **适合场景** | 自动化软件开发、需求→代码全流程 |
| **模型支持** | 多模型 |
### 8. DeerFlow（字节跳动）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | SuperAgent Harness——从 Deep Research 框架演化为通用超级 Agent 运行时 |
| **核心特性** | 多 Agent 协调（Planner + Worker）、沙箱代码执行、长期记忆、MCP 工具集成、深度研究能力 |
| **优势** | 极高人气（\~79K stars）；安全沙箱执行；内置记忆系统；从研究到代码到创作一体化 |
| **劣势** | 偏研究/内容生产场景；相对新，生产级案例待验证 |
| **适合场景** | 深度研究报告、自动代码生成、内容创作、长链路自主任务 |
| **模型支持** | 多模型（通过 MCP 协议接入） |
### 9. DeepAgent
> 注意：DeepAgent 有两个含义——①学术论文 "DeepAgent: A General Reasoning Agent with Scalable Toolsets"（WWW 2026），②Abacus AI 平台的自主 Agent 产品。这里分别总结。
#### 9a. DeepAgent（学术框架）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 通用推理 Agent + 可扩展工具集，支持动态工具发现与组合 |
| **核心特性** | 工具集可自动扩展、通用推理链、多步规划执行 |
| **优势** | 学术前沿；工具扩展性强；通用推理能力出色 |
| **劣势** | 偏学术研究，生产就绪度较低 |
| **适合场景** | 研究探索、需要动态工具组合的复杂推理任务 |
| **论文** | arXiv:2510.21618（2025年10月） |
#### 9b. DeepAgent（Abacus AI 产品）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 全能型自主 Agent 平台，执行复杂任务而非仅生成回答 |
| **核心特性** | 自主构建应用、部署工作流、always-on Agent、多模型路由 |
| **优势** | 全栈自主能力（研究→编码→部署）；集成 100+ 模型；支持自动化工作流 |
| **劣势** | 闭源商业产品；定价较高 |
| **适合场景** | 全栈应用自动化构建、企业工作流自动化 |
| **模型支持** | 100+ 模型（GPT、Claude、Gemini、DeepSeek 等） |
### 10. ModelScope-Agent（阿里巴巴 / 魔搭）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 基于开源 LLM 构建可定制 Agent 系统，工具注册 + 多模型适配 |
| **核心特性** | 工具注册机制、RAG 集成、多模型适配（Qwen 系列优先）、ModelScope 模型生态 |
| **优势** | 与阿里开源模型生态深度集成；支持 Qwen 全系列；中文场景优化好 |
| **劣势** | 国际社区影响力有限；文档和教程以中文为主 |
| **适合场景** | 基于 Qwen/开源模型的 Agent 开发、中文场景、国内企业应用 |
| **模型支持** | Qwen 全系列 + 魔搭社区模型 + 通用 API |
### 11. Dify
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 低代码可视化工作流平台，拖拽式定义 Agent |
| **核心特性** | 可视化工作流编辑器、RAG 管线、Agent + Workflow 三合一、开源自部署 |
| **优势** | 非开发者可上手；多模型接入；开源可私有化部署 |
| **劣势** | 复杂逻辑受限于可视化表达；定制化上限较低 |
| **适合场景** | 快速搭建业务 Agent、非技术团队、企业内部工具 |
| **模型支持** | 多模型（OpenAI、Anthropic、开源等） |
### 12. Coze（字节跳动 / 扣子）
| 维度 | 说明 |
| --- | --- |
| **架构思想** | 面向消费者的低代码 Bot/Agent 搭建平台 |
| **核心特性** | 可视化 Bot 搭建、插件市场、多渠道发布（飞书/微信/Discord） |
| **优势** | 极低门槛；内置丰富插件；多平台一键发布 |
| **劣势** | 灵活性有限；深度定制困难；依赖平台生态 |
| **适合场景** | 个人/中小企业快速搭建 Bot、内容创作助手 |
| **模型支持** | 豆包模型 + 第三方模型 |
---
## 三、架构模式对比
| 模式 | 代表框架 | 核心理念 | 优势 | 劣势 |
| --- | --- | --- | --- | --- |
| **图（Graph）模型** | LangGraph, Google ADK | 显式定义状态机/DAG | 精确可控、可调试、Token 高效 | 学习成本高、简单场景繁琐 |
| **角色（Crew）模型** | CrewAI, MetaGPT | 定义角色分工，框架调度 | 直觉化、上手快 | 控制粒度粗、调试困难 |
| **对话（Chat）模型** | AutoGen | Agent 间自由对话协商 | 灵活、适合探索性任务 | Token 消耗大、结果不确定 |
| **Harness 模型** | DeerFlow, Claude Code | 围绕 LLM 构建执行壳 | 端到端自主、深度任务处理 | 长链路可控性挑战 |
| **工具扩展模型** | DeepAgent, ModelScope-Agent | 动态工具发现与组合 | 通用性强、可扩展 | 工具质量依赖注册 |
| **低代码模型** | Dify, Coze | 可视化拖拽编排 | 门槛极低、快速上线 | 灵活性天花板低 |
---
## 四、性能与效率对比
基于公开 benchmark 数据：
| 指标 | LangGraph | CrewAI | AutoGen | 备注 |
| --- | --- | --- | --- | --- |
| **执行速度** | 基准 | 慢 2.2x | 慢 8-9x | 相同任务对比 |
| **Token 效率** | 高 | 中 | 低 | 对话开销差距显著 |
| **冷启动** | 快 | 中 | 慢 | 框架初始化开销 |
| **生产稳定性** | 高（1.0 API） | 中高 | 中（API 变动频繁） | 2026 年评估 |
---
## 五、选型决策矩阵
| 你的场景 | 推荐框架 | 备选 |
| --- | --- | --- |
| 精确控制工作流 + 生产部署 | **LangGraph** | Google ADK |
| 多角色团队协作（研究/内容） | **CrewAI** | MetaGPT |
| Agent 间需要讨论辩论 | **AutoGen** | — |
| 快速原型 + OpenAI 生态 | **OpenAI Agents SDK** | — |
| Google Cloud 原生 | **Google ADK** | — |
| 模拟软件开发 SOP 流程 | **MetaGPT** | — |
| 深度研究 + 长链路自主任务 | **DeerFlow** | — |
| 通用推理 + 动态工具集 | **DeepAgent** | — |
| 基于 Qwen/国产开源模型 | **ModelScope-Agent** | — |
| 低代码/可视化/业务快速落地 | **Dify** | Coze |
| 企业级 .NET/Java 集成 | **Semantic Kernel** | — |
| 面向消费者 Bot | **Coze** | Dify |
---
## 六、协议与互操作标准
| 协议 | 说明 | 支持框架 |
| --- | --- | --- |
| **MCP (Model Context Protocol)** | Anthropic 提出的工具/上下文标准协议 | DeerFlow、LangGraph、Claude Agent SDK、大多数主流框架 |
| **A2A (Agent-to-Agent)** | Google 提出的 Agent 间通信协议 | Google ADK、部分框架适配中 |
| **OpenAI Function Calling** | OpenAI 的工具调用标准 | OpenAI SDK、LangGraph、CrewAI 等 |
---
## 七、趋势总结
1. **协议标准化加速**：MCP 和 A2A 正在成为行业事实标准，框架间互操作性大幅提升
2. **从实验到生产**：2025-2026 年各框架纷纷发布稳定版（LangGraph 1.0、CrewAI checkpoint），企业采用进入生产阶段
3. **大厂全面入局**：OpenAI（Agents SDK）、Google（ADK）、微软（AutoGen/SK）、字节（DeerFlow/Coze）、阿里（ModelScope-Agent）均有布局
4. **SuperAgent Harness 兴起**：DeerFlow（79K stars）引领的"超级 Agent 运行时"模式，将研究、编码、创作融为一体
5. **效率差距显著**：不同框架间 Token 效率差距可达 8-9 倍，选型时需重点关注成本
6. **混合编排成主流**：单一模式（纯图/纯角色/纯对话）各有局限，混合模式（如 Google ADK）成为发展方向
7. **安全沙箱成标配**：代码执行安全隔离从"可选"变为"必须"，DeerFlow、OpenAI SDK 均内置沙箱
8. **记忆与长期状态**：长期记忆系统从附加功能变为核心特性，支持跨会话上下文保持
---
## 八、快速对比图（按维度评分）
> 评分标准：⭐⭐⭐⭐⭐ = 该维度极强
| 框架 | 易用性 | 灵活性 | 生产就绪 | 多Agent | 生态 | 社区 |
| --- | --- | --- | --- | --- | --- | --- |
| LangGraph | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| CrewAI | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| AutoGen | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| OpenAI SDK | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Google ADK | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Semantic Kernel | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| MetaGPT | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
| DeerFlow | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| DeepAgent | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| ModelScope-Agent | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| Dify | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Coze | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ |
---
## 九、参考资源
- LangGraph: [https://github.com/langchain-ai/langgraph](https://github.com/langchain-ai/langgraph)
- CrewAI: [https://github.com/crewAIInc/crewAI](https://github.com/crewAIInc/crewAI)
- AutoGen: [https://github.com/microsoft/autogen](https://github.com/microsoft/autogen)
- OpenAI Agents SDK: [https://github.com/openai/openai-agents-python](https://github.com/openai/openai-agents-python)
- Google ADK: [https://github.com/google/adk-python](https://github.com/google/adk-python)
- Semantic Kernel: [https://github.com/microsoft/semantic-kernel](https://github.com/microsoft/semantic-kernel)
- MetaGPT: [https://github.com/geekan/MetaGPT](https://github.com/geekan/MetaGPT)
- DeerFlow: [https://github.com/bytedance/deer-flow](https://github.com/bytedance/deer-flow)
- DeepAgent 论文: [https://arxiv.org/abs/2510.21618](https://arxiv.org/abs/2510.21618)
- ModelScope-Agent: [https://github.com/modelscope/modelscope-agent](https://github.com/modelscope/modelscope-agent)
- Dify: [https://github.com/langgenius/dify](https://github.com/langgenius/dify)
- Coze: [https://www.coze.com](https://www.coze.com/)