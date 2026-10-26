# AI 应用落地全景
## 全景架构
```plain text
┌─────────────────────────────────────────────────────────────────┐
│                      应用层（Application）                       │
│         Chatbot / Copilot / Agent / 自动化工作流                 │
├─────────────────────────────────────────────────────────────────┤
│                      能力层（Capability）                        │
│     RAG / GraphRAG / Tool Use / Multi-Agent / Planning          │
├─────────────────────────────────────────────────────────────────┤
│                      工程层（Engineering）                       │
│   Prompt / Data Pipeline / Vector DB / Eval / Guardrail / Obs   │
├─────────────────────────────────────────────────────────────────┤
│                      模型层（Model）                             │
│        Foundation Model / Fine-tune / Embedding / Reranker      │
└─────────────────────────────────────────────────────────────────┘
```
每一层解决不同的问题，往上走离业务越近，往下走离基础设施越近。
---
## 一、模型层 — "大脑"
> **核心问题：** AI 的基础推理和生成能力从哪来？
### 1.1 基座模型直接调用
- 通过 API 调用大模型，zero-shot / few-shot 完成任务
- 适用于通用任务和快速验证阶段
- 代表：Claude、GPT、Qwen、DeepSeek
### 1.2 Prompt Engineering
- 通过设计提示词引导模型行为，是大多数场景的第一步
- 核心技术：
	- **Zero-shot**：直接提问，不给示例
	- **Few-shot**：给几个示例，让模型学习输入/输出模式
	- **Chain-of-Thought（CoT）**：引导模型分步推理
	- **System Prompt**：设定模型角色、约束和输出格式
### 1.3 Fine-tuning（微调）
- 用领域数据微调模型参数，适应特定任务/格式/风格
- 适用于 Prompt 搞不定的场景（领域术语多、输出格式严格、需要特定风格）
- 技术路线：
	- **LoRA / QLoRA**：低成本参数高效微调，主流方案
	- **全量微调**：效果最好但成本高，需要大量数据和算力
	- **RLHF / DPO**：基于人类反馈的对齐训练
### 1.4 Embedding 模型
- 将文本映射为高维向量，支撑语义检索
- 代表：BGE、GTE、OpenAI text-embedding、Cohere Embed
### 1.5 Reranker（重排序模型）
- 对初检结果做精细化重排序，提升检索精度
- 代表：bge-reranker、Cohere Rerank、cross-encoder
### 1.6 关键决策路径
```plain text
先用 Prompt 调通 → 不行上 RAG → 还不行再 Fine-tune
90% 的场景不需要微调
```
---
## 二、工程层 — "骨骼和血管"
> **核心问题：** 如何让模型能力在生产环境中可靠运行？
### 2.1 数据工程
#### 数据处理流水线
```plain text
原始数据（文档 / 数据库 / API / 网页）
    ↓
数据采集与清洗（去重、格式统一、质量过滤）
    ↓
文档解析（PDF / Word / HTML / 表格识别 / OCR）
    ↓
文本分块（Chunking）
    ↓
向量化（Embedding）
    ↓
存储（向量数据库 + 元数据）
```
#### 文档解析
- **解决的问题：** 非结构化数据变结构化
- **工具：** Unstructured、LlamaParse、Docling、Marker（PDF专用）
#### 分块策略（Chunking）
- **解决的问题：** 检索粒度与上下文完整性的平衡
- **常见策略：**
	- **固定长度分块**：简单直接，按 token 数切分
	- **递归字符分块**：按段落 → 句子 → 字符逐级切分
	- **语义分块**：基于语义相似度判断边界
	- **文档结构分块**：按标题、章节等结构切分
- **工具：** LangChain Splitters、LlamaIndex Node Parsers
#### 向量数据库
- **解决的问题：** 对海量向量进行高效近似最近邻（ANN）检索
- **选型：**
	- **Milvus**：分布式，适合大规模生产环境
	- **Qdrant**：Rust 实现，性能好，API 友好
	- **Chroma**：轻量级，适合原型开发
	- **FAISS**：Meta 开源库，纯计算层，需要自己管理存储
	- **Weaviate**：内置向量化能力，GraphQL 查询
#### 元数据管理
- 与向量库配合，存储 source / date / author / permission 等
- 支持过滤检索、权限控制、版本追溯
### 2.2 评估体系（Eval）
> **解决的问题：** 怎么知道 AI 输出质量？怎么持续迭代？
#### 离线评估
- **准确性**：与标准答案对比（Exact Match、F1）
- **忠实度（Faithfulness）**：回答是否基于检索到的内容，而非编造
- **相关性（Relevance）**：检索结果是否与问题相关
- **有害性检测**：是否包含不当内容
#### 在线评估
- **用户反馈**：点赞/点踩、评分、纠正
- **使用指标**：使用率、留存率、会话完成率
- **质量指标**：错误率、幻觉率、拒答率
- **性能指标**：延迟（TTFT / TPS）、成本（token 消耗）
#### 评估工具
- **RAGAS**：RAG 管线专用评估（忠实度、答案相关性、上下文精度）
- **DeepEval**：通用 LLM 评估框架
- **Promptfoo**：Prompt 回归测试，CI/CD 集成
- **人工标注**：黄金标准，尤其在早期阶段不可替代
### 2.3 安全护栏（Guardrail）
> **解决的问题：** 防止模型说不该说的话、做不该做的事
#### 输入侧
- **Prompt 注入检测**：识别并拦截试图绕过指令的恶意输入
- **话题边界**：只回答业务相关问题，拒绝越界请求
- **PII 脱敏**：自动识别并脱敏个人隐私信息
- **权限校验**：确认用户有权访问对应知识/功能
#### 输出侧
- **内容审核**：敏感信息过滤、合规检查
- **幻觉检测**：输出内容是否有检索依据
- **格式校验**：JSON Schema、结构化输出验证
- **引用溯源**：标注回答来自哪个原始文档
#### 工具
- Guardrails AI、NeMo Guardrails（NVIDIA）、自定义规则引擎
### 2.4 可观测性（Observability）
> **解决的问题：** 线上出了问题能快速定位
#### 一次请求的完整链路追踪
```plain text
User Query → Guardrail → Retrieval → Rerank → Prompt Assembly → LLM → Post-process → Response
   ↓            ↓           ↓          ↓            ↓              ↓        ↓           ↓
  日志         耗时        召回数     分数分布     token 数       延迟     格式检查     用户反馈
```
#### 关键指标
- **延迟**：首 token 时间（TTFT）、每秒 token 数（TPS）、端到端延迟
- **质量**：检索命中率、重排序分数分布、幻觉率
- **成本**：token 消耗、API 调用次数、每次查询成本
- **错误**：失败率、超时率、格式错误率
#### 工具
- **LangSmith**：LangChain 配套，Trace 可视化
- **Langfuse**：开源，支持多框架
- **Phoenix（Arize）**：开源，Trace + Eval 一体化
- **OpenTelemetry**：通用可观测标准，可集成到现有监控体系
---
## 三、能力层 — "技能树"
> **核心问题：** 模型的基础能力如何组装成具体的 AI 能力？
### 3.1 知识增强系列 — 让 AI 懂领域知识
#### 问题本质
模型不知道你的私有数据，训练数据有截止日期，会产生幻觉。
#### 方案光谱（从简单到复杂）
```plain text
Prompt 塞知识  →  基础 RAG  →  高级 RAG  →  GraphRAG  →  知识图谱 + 推理
```
#### 方案详解
**Prompt 内嵌知识**
- 直接把关键信息写进 System Prompt
- 适用：知识量小（\< 10 页）
- 局限：上下文窗口有限，知识更新需要改代码
**基础 RAG（Retrieval-Augmented Generation）**
- 流程：用户提问 → 向量检索相似文档片段 → 拼进 Prompt → LLM 生成
- 适用：大多数知识问答场景
- 局限：只能局部匹配，无法全局推理，对检索质量强依赖
**高级 RAG**
- 在基础 RAG 上加多种优化手段：
```plain text
查询阶段优化：
├── Query Rewriting — 改写用户问题（修正模糊表述）
├── HyDE — 先生成假设答案，再用答案去检索
├── Query Decomposition — 复杂问题拆成子问题逐个检索
└── Step-back Prompting — 先问更抽象的问题获取背景知识

检索阶段优化：
├── 混合检索 — 向量检索 + 关键词检索（BM25）互补
├── 多路召回 — 不同分块策略 / 不同索引并行检索
├── 父子文档 — 检索细粒度块，返回时扩展到父级块
└── 元数据过滤 — 按时间 / 来源 / 权限缩小范围

生成阶段优化：
├── Reranker 重排序 — 精排检索结果
├── 上下文压缩 — 去掉不相关段落，节省 token
├── 引用溯源 — 标注答案来自哪个文档哪一段
└── Self-RAG — 模型自主判断是否需要检索、检索结果是否有用
```
**GraphRAG**
- 流程：文档 → 实体/关系抽取 → 构建知识图谱 → 社区检测 → 分层摘要 → 图结构检索 + LLM 生成
- 两种检索模式：
	- **Local Search**：从实体出发，扩展邻域子图（回答具体问题）
	- **Global Search**：Map-Reduce 遍历社区摘要（回答全局性问题）
- 适用：跨文档推理、全局性总结
- 局限：索引成本高，需要较多 LLM 调用
**知识图谱 + 规则推理**
- 结构化本体建模（OWL/RDF） + 逻辑规则引擎
- 适用：合规、医疗、金融等强逻辑、强一致性领域
- 局限：构建和维护成本最高，需要领域专家参与
#### 知识增强选型指南
| 维度 | Prompt 内嵌 | 基础 RAG | 高级 RAG | GraphRAG | 知识图谱 |
| --- | --- | --- | --- | --- | --- |
| 知识规模 | \< 10 页 | 中等 | 大 | 大 | 大 |
| 实现复杂度 | 低 | 中 | 高 | 高 | 最高 |
| 全局推理能力 | 无 | 弱 | 中 | 强 | 最强 |
| 实时性 | 改代码 | 更新索引 | 更新索引 | 重建图 | 更新图 |
| 适用场景 | FAQ | 文档问答 | 复杂问答 | 跨文档分析 | 强逻辑领域 |
### 3.2 工具使用系列 — 让 AI 操作外部世界
#### 问题本质
模型只能生成文本，不能执行动作、不能访问实时数据。
#### Function Calling（函数调用）
- 模型根据用户意图，决定调用哪个函数、传什么参数
- 宿主程序执行函数并将结果返回给模型
- 示例：查天气、查数据库、调业务 API
#### MCP（Model Context Protocol）
- Anthropic 提出的标准化工具连接协议
- 核心价值：一次接入，多模型复用；工具生态标准化
- 架构：Client（AI 应用） ↔ Server（工具提供方）
- 支持：Tools（工具调用）、Resources（上下文资源）、Prompts（预定义模板）
#### Code Interpreter（代码解释器）
- LLM 生成代码并在沙箱中执行
- 适用：数据分析、画图、文件处理、数学计算
- 代表：OpenAI Code Interpreter、Open Interpreter
#### 工具使用的关键设计
```plain text
工具描述要清晰 — 模型根据描述选择工具，描述质量直接影响调用准确率
参数 Schema 要严格 — 用 JSON Schema 约束参数类型和格式
错误处理要完善 — 工具调用可能失败，需要模型能理解错误并重试或换策略
权限控制要到位 — 不是所有工具都应该让模型随意调用
```
### 3.3 Agent 系列 — 让 AI 自主完成复杂任务
#### 问题本质
单轮对话解决不了多步骤、需要动态决策的任务。
#### Agent 核心公式
```plain text
Agent = LLM（推理引擎）+ Planning（规划）+ Memory（记忆）+ Tools（工具）
```
#### 运行循环
```plain text
感知（理解任务和当前状态）
  → 规划（拆解步骤 / 选择下一步）
    → 行动（调用工具执行）
      → 观察（看执行结果）
        → 反思（评估是否需要调整策略）
          → 继续 or 结束
```
#### Agent 设计模式
**ReAct（Reasoning + Acting）**
- 思考和行动交替进行，每一步都有推理过程
- 灵活通用，适合步骤不确定的探索性任务
**Plan-and-Execute**
- 先做完整计划，再逐步执行，执行中可修订计划
- 适合步骤相对明确、需要全局视角的任务
**Multi-Agent（多智能体协作）**
- 多个 Agent 各有分工，通过消息传递协作
- 模式：
	- **主从模式**：一个 Orchestrator 分配任务给专业 Agent
	- **对话模式**：Agent 之间互相讨论/辩论
	- **流水线模式**：任务在 Agent 之间按顺序流转
- 适合复杂项目、需要多角色多视角的场景
**Human-in-the-loop**
- 关键节点暂停，等人类确认后再继续
- 适合高风险操作（如：删除数据、发布上线、发送邮件）
#### Agent 框架
- **LangGraph**：基于图的 Agent 编排，LangChain 生态
- **CrewAI**：Multi-Agent 框架，角色扮演风格
- **AutoGen**：微软开源，多 Agent 对话框架
- **Dify / Coze**：低代码 Agent 平台
### 3.4 记忆系列 — 让 AI 有持续上下文
#### 问题本质
对话上下文有限，跨会话遗忘，无法积累和利用历史交互信息。
#### 记忆分类
```plain text
短期记忆（Working Memory）
├── 当前对话历史
├── 滑动窗口（只保留最近 N 轮）
└── 摘要压缩（用 LLM 对历史对话生成摘要）

长期记忆（Long-term Memory）
├── 用户画像（角色、偏好、知识水平）
├── 历史决策和反馈
├── 存储方式：向量数据库 / 结构化数据库
└── 检索方式：语义检索 + 时间衰减

工作记忆（Scratchpad）
├── 当前任务的中间状态
├── 推理过程的临时记录
└── 执行计划和进度追踪
```
---
## 四、应用层 — "最终产品形态"
> **核心问题：** AI 能力如何包装成用户能用的产品？
### 4.1 产品形态光谱
```plain text
自动化程度    低 ──────────────────────────────────────── 高

             Chatbot    Copilot    Agent      Workflow
             对话机器人  辅助助手   自主代理    全自动流程

人类角色      主导       主导       监督        审批
AI 角色       回答       建议       执行        端到端执行
```
### 4.2 各形态详解
#### Chatbot（对话机器人）
- **场景**：客服、知识问答、FAQ
- **能力组合**：RAG + Guardrail
- **特点**：被动响应，一问一答，最容易落地
#### Copilot（辅助助手）
- **场景**：编程助手、写作助手、数据分析助手
- **能力组合**：RAG + Tool Use + 上下文理解
- **特点**：嵌入用户工作流，给建议而非直接执行，人类保持控制权
#### Agent（自主代理）
- **场景**：自动化数据分析、运维排障、代码审查
- **能力组合**：Planning + Tool Use + Memory + 自我反思
- **特点**：接到任务后自主规划和执行，关键节点可人工确认
#### 自动化工作流
- **场景**：定时报告生成、数据处理管线、批量内容生产
- **能力组合**：Multi-Agent + Tool Use + Scheduler + 触发器
- **特点**：全自动运行，人类只做审批和异常处理
---
## 五、决策地图
根据你面对的问题，选择合适的技术方案：
```plain text
你的问题是什么？
│
├── AI 不懂我的业务知识
│   ├── 知识量小（< 10 页） → System Prompt 内嵌
│   ├── 中等规模文档问答 → 基础 RAG
│   ├── 复杂问答 / 多跳推理 → 高级 RAG
│   ├── 跨文档全局理解 → GraphRAG
│   └── 强逻辑推理（合规/医疗） → 知识图谱 + 规则引擎
│
├── AI 需要执行外部操作
│   ├── 调 API / 查数据库 → Function Calling
│   ├── 多工具标准化接入 → MCP 协议
│   └── 数据分析 / 跑代码 → Code Interpreter
│
├── 任务太复杂，需要多步骤
│   ├── 步骤明确 → Plan-and-Execute
│   ├── 步骤不确定 → ReAct Agent
│   └── 需要多角色协作 → Multi-Agent
│
├── 输出质量不可控
│   ├── 幻觉问题 → RAG + 引用溯源 + 忠实度检测
│   ├── 安全问题 → Guardrail（输入 / 输出侧）
│   └── 不稳定 → Eval 体系 + Prompt 回归测试
│
└── 需要持续迭代优化
    ├── 线上问题定位 → Tracing + Logging（可观测性）
    ├── 质量度量 → 离线 Eval + 在线用户反馈
    └── 数据飞轮 → 用户反馈 → 标注 → 优化检索 / Prompt / 模型
```
---
## 六、技术栈速查
| 层次 | 方向 | 代表工具 / 框架 |
| --- | --- | --- |
| 模型层 | 基座模型 | Claude、GPT、Qwen、DeepSeek |
| 模型层 | 微调 | LoRA、QLoRA、LLaMA-Factory |
| 模型层 | Embedding | BGE、GTE、OpenAI Embedding |
| 工程层 | 文档解析 | Unstructured、LlamaParse、Docling |
| 工程层 | 向量数据库 | Milvus、Qdrant、Chroma、FAISS |
| 工程层 | 评估 | RAGAS、DeepEval、Promptfoo |
| 工程层 | 护栏 | Guardrails AI、NeMo Guardrails |
| 工程层 | 可观测 | LangSmith、Langfuse、Phoenix |
| 能力层 | RAG 框架 | LangChain、LlamaIndex |
| 能力层 | GraphRAG | Microsoft GraphRAG、nano-graphrag |
| 能力层 | 知识图谱 | Neo4j、Apache Jena、NetworkX |
| 能力层 | Agent 框架 | LangGraph、CrewAI、AutoGen |
| 能力层 | 工具协议 | MCP、Function Calling |
| 应用层 | 低代码平台 | Dify、Coze、FastGPT |
| 应用层 | 开发框架 | LangChain、LlamaIndex、Semantic Kernel |