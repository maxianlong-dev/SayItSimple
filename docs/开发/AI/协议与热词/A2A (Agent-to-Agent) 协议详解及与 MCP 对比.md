# A2A (Agent-to-Agent) 协议详解及与 MCP 对比
## 一、A2A 是什么
### 1.1 一句话定义
A2A (Agent-to-Agent Protocol) 是 Google 于 2025 年 4 月发布的开放协议，定义了 AI Agent 之间如何发现、通信和协作的标准——类似 Agent 世界的 "HTTP 协议"。
### 1.2 解决什么问题
**现状痛点：**
- 不同框架构建的 Agent 无法互相通信（LangChain Agent 调不了 CrewAI Agent）
- 企业内部多个 Agent 形成孤岛，无法协作完成跨域任务
- 没有标准的 Agent 能力描述和发现机制
- Agent 间协作缺乏统一的任务委托、状态同步、结果交付规范
**A2A 解决：**
- 跨框架、跨平台的 Agent 互操作
- 标准化的 Agent 能力发现（Agent Card）
- 统一的任务生命周期管理
- 支持长时间运行任务的异步协作
### 1.3 核心思想
```plain text
Agent A (Client)                    Agent B (Server/Remote Agent)
     |                                        |
     |—— 发现 Agent Card ——————————————→     |  （了解能力）
     |                                        |
     |—— 发送 Task ——————————————————→      |  （委托任务）
     |                                        |
     |←—— 状态更新 / 流式结果 ————————       |  （进度同步）
     |                                        |
     |←—— 最终 Artifact ———————————————     |  （交付成果）
```
**本质：A2A 是 Agent 之间的"委托与协作"协议。**
---
## 二、A2A 核心概念
### 2.1 Agent Card（代理名片）
Agent 的能力描述文件，托管在 `/.well-known/agent.json`，类似网站的 robots.txt。
```json
{
  "name": "Travel Planner Agent",
  "description": "Plans travel itineraries including flights, hotels, and activities",
  "url": "https://travel-agent.example.com",
  "version": "1.0.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "plan_trip",
      "name": "Plan Trip",
      "description": "Create a complete travel itinerary given destination and dates",
      "inputModes": ["text"],
      "outputModes": ["text", "file"]
    }
  ],
  "authentication": {
    "schemes": ["OAuth2", "ApiKey"]
  }
}
```
**关键字段：**
- `skills`：Agent 能做什么（可被发现和选择）
- `capabilities`：支持的协议特性（流式、推送等）
- `authentication`：鉴权方式
- `inputModes/outputModes`：支持的数据格式
### 2.2 Task（任务）
A2A 通信的核心单元，表示一次委托请求。
**Task 生命周期：**
```plain text
submitted → working → completed
                  ↘ failed
                  ↘ canceled
                  ↘ input-required（需要补充信息）
```
**Task 结构：**
```json
{
  "id": "task-123",
  "status": {
    "state": "working",
    "message": "Searching for flights..."
  },
  "artifacts": [],
  "history": [
    {
      "role": "user",
      "parts": [{"type": "text", "text": "Plan a 5-day trip to Tokyo"}]
    }
  ]
}
```
### 2.3 Artifact（制品）
Task 的输出产物，可以是文本、文件、结构化数据等。
```json
{
  "name": "itinerary",
  "parts": [
    {"type": "text", "text": "Day 1: Arrive at Narita..."},
    {"type": "file", "file": {"mimeType": "application/pdf", "data": "base64..."}}
  ]
}
```
### 2.4 Message 与 Part
- **Message**：对话中的一条消息（user 或 agent 角色）
- **Part**：消息的组成部分（TextPart、FilePart、DataPart）
- 支持多模态：文本、文件、结构化数据混合
### 2.5 核心 API 方法
| 方法 | 说明 |
| --- | --- |
| `tasks/send` | 发送/更新任务（同步响应） |
| `tasks/sendSubscribe` | 发送任务并订阅 SSE 流式更新 |
| `tasks/get` | 查询任务状态 |
| `tasks/cancel` | 取消任务 |
| `tasks/pushNotification/set` | 设置 Webhook 推送 |
| `tasks/pushNotification/get` | 查询推送配置 |
| `tasks/resubscribe` | 重新订阅断开的流 |
---
## 三、A2A 工作流程
### 3.1 典型交互流程
```plain text
1. Client 获取 Remote Agent 的 Agent Card
2. Client 根据 skills 选择合适的 Agent
3. Client 发送 tasks/send 或 tasks/sendSubscribe
4. Remote Agent 开始处理：
   a. 状态变为 working
   b. 如需补充信息 → 状态变为 input-required → Client 补充后继续
   c. 流式推送中间结果（streaming）
5. 处理完成 → 状态变为 completed，返回 Artifacts
6. Client 获取最终结果
```
### 3.2 多轮交互（Input Required）
```plain text
Client: "帮我订机票"
Agent: [input-required] "请问出发城市和目的地是？"
Client: "北京到东京，下周一出发"
Agent: [working] → [completed] + Artifact
```
这是 A2A 的亮点——支持 Agent 间的多轮协商。
### 3.3 长时间任务
```plain text
Client → tasks/send → Agent 返回 task-id + status: submitted
Client → tasks/pushNotification/set(webhook_url)
... Agent 在后台处理 ...
Agent → POST webhook_url（推送状态变更）
Client → tasks/get(task-id) → 获取最终结果
```
---
## 四、A2A vs MCP 核心对比
### 4.1 定位差异（最关键的区别）
| 维度 | MCP | A2A |
| --- | --- | --- |
| 连接对象 | LLM ↔ 工具/数据 | Agent ↔ Agent |
| 类比 | 人使用工具（锤子、电脑） | 人委托别人做事 |
| 角色关系 | 主从关系（LLM 调用工具） | 对等关系（Agent 间协作） |
| 智能位置 | 只有 Client 端有 LLM | 两端都可能有 LLM |
**一句话总结：MCP 让 Agent 能"使用工具"，A2A 让 Agent 能"雇佣其他 Agent"。**
### 4.2 详细对比
| 维度 | MCP | A2A |
| --- | --- | --- |
| **发起者** | Anthropic (2024.11) | Google (2025.04) |
| **协议层** | 应用 ↔ 工具 | Agent ↔ Agent |
| **通信方式** | JSON-RPC 2.0 | HTTP + JSON-RPC |
| **传输层** | stdio / HTTP+SSE / Streamable HTTP | HTTP（REST-like + SSE） |
| **能力发现** | `tools/list`、`resources/list` | Agent Card (`/.well-known/agent.json`) |
| **核心原语** | Tools、Resources、Prompts | Tasks、Artifacts、Messages |
| **状态管理** | 无状态（单次调用） | 有状态（Task 生命周期） |
| **异步支持** | 有限（progress 通知） | 原生支持（长任务、Webhook推送） |
| **多轮交互** | 不支持（工具调用是单次的） | 原生支持（input-required 状态） |
| **执行控制** | Client 完全控制 | Remote Agent 自主决策 |
| **透明度** | 工具执行过程对 Client 可见 | Agent 内部过程对外不透明（黑盒） |
| **适用规模** | 单 Agent 增强 | 多 Agent 编排 |
### 4.3 协作模式对比
**MCP 模式：**
```plain text
User → Agent → [MCP] → Tool1（执行并返回）
                     → Tool2（执行并返回）
                     → Tool3（执行并返回）
Agent 组装结果 → User
```
Agent 是唯一的"大脑"，工具只是手脚。
**A2A 模式：**
```plain text
User → Agent A → [A2A] → Agent B（自主完成子任务）
                       → Agent C（自主完成子任务）
Agent A 综合结果 → User
```
每个 Agent 都是独立的"大脑"，自主规划和执行。
### 4.4 互补关系
**MCP 和 A2A 不是竞争关系，而是互补的两层：**
```plain text
┌─────────────────────────────────────────┐
│           Multi-Agent System            │
│                                         │
│   Agent A ←——— A2A ———→ Agent B        │  ← Agent 间协作用 A2A
│     │                      │            │
│   [MCP]                  [MCP]          │  ← 各自用 MCP 调工具
│     │                      │            │
│  Tools/DB              APIs/Files       │
└─────────────────────────────────────────┘
```
- **对外协作**：A2A（Agent 间对话、委托、协商）
- **对内能力**：MCP（Agent 自身调用工具和数据）
---
## 五、A2A 的设计亮点
### 5.1 不透明性（Opaque Execution）
- Client 不需要知道 Remote Agent 内部怎么实现
- 可能用了 LLM，也可能是规则引擎，对外无差别
- 隐藏实现细节，降低耦合
### 5.2 框架无关
- 不规定 Agent 内部用什么框架（LangChain、CrewAI、AutoGen 都行）
- 只规定通信接口，内部实现自由
- 真正的互操作性
### 5.3 原生异步
- Task 天然支持长时间运行
- 推送通知避免轮询
- 断线重连（resubscribe）
- 适合真实业务场景（审批流、数据处理）
### 5.4 多模态支持
- Part 支持 text / file / data
- 可以传递 PDF、图片、结构化数据
- 不局限于纯文本交互
---
## 六、A2A 踩坑点与挑战
### 6.1 Agent Card 设计不当
**坑：** skills 描述不清晰，其他 Agent 无法准确判断是否合适。
**建议：**
- 每个 skill 明确输入输出格式
- description 写清楚边界（能做什么、不能做什么）
- 版本管理，向后兼容
### 6.2 任务粒度问题
**坑：** 委托的任务太大或太小。
- 太大：Remote Agent 能力不够，超时或失败
- 太小：A2A 开销大于直接 MCP 调用工具
**建议：** A2A 适合"需要推理决策"的子任务，简单确定性操作用 MCP。
### 6.3 错误传播与恢复
**坑：** Agent B 失败了，Agent A 不知道如何恢复。
**建议：**
- 利用 `status.message` 返回有意义的错误信息
- Client Agent 设计 fallback 策略（重试、换 Agent、降级）
- 设置合理超时
### 6.4 安全与信任
**坑：** 如何信任未知的 Remote Agent？
**挑战：**
- Agent Card 可以伪造能力描述
- Remote Agent 可能返回有害内容
- 敏感数据传给第三方 Agent 的风险
**建议：**
- 鉴权机制（OAuth2、API Key）
- Agent 注册与审核（企业内部白名单）
- 数据脱敏后再委托
- 结果验证（不盲信返回）
### 6.5 调试困难
**坑：** 多个 Agent 链式调用，出错难以定位。
**建议：**
- 全链路 Trace ID 透传
- 每个 Agent 记录决策日志
- 分布式追踪（类似微服务的 Tracing）
### 6.6 协议成熟度
- A2A 发布时间短（2025.04），生态尚不完善
- SDK 和工具链还在早期阶段
- 实际大规模落地案例较少
- 规范可能还会有 breaking changes
---
## 七、选型决策树
```plain text
需要让 AI 应用调用外部工具/数据？
  → 用 MCP

需要多个独立 Agent 协作完成复杂任务？
  → 用 A2A

需要 Agent 既能用工具，又能委托其他 Agent？
  → MCP + A2A 组合使用

任务是确定性的、无需推理？
  → 直接 API 调用或 MCP Tool，不需要 A2A

Agent 在同一进程/框架内？
  → 用框架内置的 multi-agent 机制（如 LangGraph）
  → 跨框架/跨组织才需要 A2A
```
---
## 八、面试高频问题
1. **A2A 和 MCP 的核心区别是什么？**
→ MCP 是 Agent-to-Tool（使用工具），A2A 是 Agent-to-Agent（委托协作）。二者互补，不竞争。
2. **为什么需要 A2A？直接 API 调用不行吗？**
→ A2A 解决的是"智能体间的动态协作"，支持多轮协商、异步长任务、能力发现。普通 API 是固定接口，无法处理开放式任务。
3. **A2A 的 Task 生命周期？**
→ submitted → working → completed/failed/canceled，中间可进入 input-required 等待补充信息。
4. **Agent Card 类比什么？**
→ 类比微服务注册中心的服务描述 + OpenAPI 文档，但面向 AI Agent 的能力发现。
5. **什么时候用 MCP 什么时候用 A2A？**
→ 确定性工具调用用 MCP，需要对方自主推理决策的子任务用 A2A。
6. **A2A 的安全模型？**
→ Agent Card 声明鉴权方式，支持 OAuth2/API Key；实际落地需企业级的 Agent 注册审核与数据脱敏。
7. **A2A 和多 Agent 框架（CrewAI、AutoGen）的关系？**
→ 框架解决进程内多 Agent 编排，A2A 解决跨框架、跨组织、跨网络的 Agent 互操作。框架可以作为 A2A Server 的内部实现。