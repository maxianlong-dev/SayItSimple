# MCP (Model Context Protocol) 知识点梳理
## 一、MCP 的本质
### 1.1 一句话定义
MCP 是 Anthropic 提出的开放协议，为 LLM 应用提供标准化的外部工具/数据接入方式——类似 AI 世界的 "USB-C 接口"。
### 1.2 解决什么问题
**没有 MCP 之前：**
- 每个 AI 应用与每个外部服务之间需要定制化集成（M×N 问题）
- 工具调用的描述格式、传参方式、鉴权方式各不相同
- 换一个 LLM 或换一个应用，集成代码全部重写
**有了 MCP 之后：**
- 工具/数据源只需实现一次 MCP Server，所有 MCP Client 都能接入
- M 个应用 + N 个服务 = M + N 次适配（而非 M × N）
- 标准化协议，工具描述、调用、响应格式统一
### 1.3 核心思想
```plain text
LLM Application (Host)
    └── MCP Client（协议客户端）
            ↕ JSON-RPC 2.0
        MCP Server（协议服务端）
            └── 对接外部工具/数据/服务
```
**本质上 MCP 是一个 C/S 架构的 RPC 协议：**
- 基于 JSON-RPC 2.0 通信
- Server 暴露 capabilities（Tools / Resources / Prompts）
- Client 发现并调用这些 capabilities
- Host 是最外层的 AI 应用（如 Claude Desktop、Cursor、Claude Code）
### 1.4 与 Function Calling 的关系
| 维度 | Function Calling | MCP |
| --- | --- | --- |
| 层级 | 模型层能力 | 应用层协议 |
| 定义方 | 应用开发者在请求中定义 | MCP Server 自行声明 |
| 发现机制 | 无，需提前硬编码 | 动态发现（list_tools） |
| 执行方 | 应用自行实现执行逻辑 | MCP Server 执行并返回结果 |
| 关系 | 底层能力 | 上层协议，底层仍依赖 Function Calling |
**MCP 不是 Function Calling 的替代品，而是建立在其之上的标准化协议层。**
---
## 二、核心架构
### 2.1 三层角色
| 角色 | 职责 | 示例 |
| --- | --- | --- |
| Host | AI 应用本身，管理 Client 生命周期 | Claude Desktop、Cursor、自研 Agent |
| Client | 协议客户端，与 Server 1:1 通信 | SDK 内置 |
| Server | 暴露工具/资源/提示词 | 文件系统 Server、GitHub Server、数据库 Server |
### 2.2 三大原语（Primitives）
**Tools（工具）**
- 模型可调用的函数/操作
- 类似 Function Calling 的 function 定义
- 例：`search_files`、`run_query`、`send_email`
- 由模型决定何时调用（model-controlled）
**Resources（资源）**
- 向模型暴露的数据/上下文
- 类似 REST 中的 GET 端点，只读
- 例：文件内容、数据库 schema、API 文档
- 由应用决定何时获取（application-controlled）
**Prompts（提示词模板）**
- 预定义的交互模板
- 可参数化，引导用户/模型完成特定任务
- 例：代码审查模板、SQL 生成模板
- 由用户选择使用（user-controlled）
### 2.3 传输层
**stdio（标准输入输出）**
- Client 启动 Server 为子进程
- 通过 stdin/stdout 通信
- 适合本地工具、CLI 集成
- 零网络开销，安全性好
**HTTP + SSE（Server-Sent Events）**
- Client 通过 HTTP POST 发送请求
- Server 通过 SSE 推送响应/通知
- 适合远程服务、多用户共享
**Streamable HTTP（2025 新增）**
- 统一端点，支持请求-响应和流式
- 逐步替代 SSE 方案
- 支持无状态和有状态两种模式
### 2.4 协议生命周期
```plain text
1. Initialize：Client → Server 握手，交换协议版本和 capabilities
2. Initialized：确认连接建立
3. 正常通信：
   - tools/list → 发现工具
   - tools/call → 调用工具
   - resources/list → 发现资源
   - resources/read → 读取资源
   - prompts/list → 发现提示词模板
4. Shutdown：优雅关闭连接
```
---
## 三、MCP Server 开发
### 3.1 基本结构（Python SDK 示例）
```python
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("my-server")

@server.list_tools()
async def list_tools():
    return [
        Tool(
            name="search",
            description="Search for documents",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search query"}
                },
                "required": ["query"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "search":
        results = do_search(arguments["query"])
        return [TextContent(type="text", text=results)]
```
### 3.2 工具描述设计原则
- **名称**：动词 + 名词，清晰表达功能（`search_issues`、`create_file`）
- **description**：告诉模型什么时候该用这个工具，不要太短也不要太长
- **inputSchema**：JSON Schema 格式，字段加 description
- **避免歧义**：多个工具功能相似时，description 中说明区别
### 3.3 常用 SDK
| 语言 | SDK |
| --- | --- |
| Python | `mcp` (官方) |
| TypeScript | `@modelcontextprotocol/sdk` (官方) |
| Java/Kotlin | Spring AI MCP、社区实现 |
| Go | `mark3labs/mcp-go` 等社区实现 |
---
## 四、常见踩坑点
### 4.1 工具描述质量直接决定调用成功率
**坑：** 工具描述写得模糊或太简略，模型不知道什么时候该调用。
**正确做法：**
- description 中明确说明使用场景和限制
- 参数 description 说明格式要求（日期格式、ID 格式等）
- 提供 enum 约束可选值
- 负面说明：不适合用这个工具的情况
```json
{
  "name": "query_orders",
  "description": "Query orders by date range. Use this when user asks about order history or order status. Do NOT use for creating or modifying orders.",
  "inputSchema": {
    "properties": {
      "start_date": {
        "type": "string",
        "description": "Start date in YYYY-MM-DD format"
      }
    }
  }
}
```
### 4.2 工具数量过多导致选择困难
**坑：** 注册了几十个工具，模型选错或漏选。
**解决方案：**
- 动态工具加载：根据上下文只暴露相关工具
- 工具分层：先用一个"路由工具"确定类别，再加载具体工具
- 合并相似工具：用参数区分而非多个工具
- 设置优先级或分组
### 4.3 stdio 模式下的环境问题
**坑：** Server 子进程环境变量、路径与预期不同。
**常见症状：**
- `command not found`（PATH 不对）
- 找不到配置文件（工作目录不对）
- 权限问题
**解决方案：**
- 在 MCP 配置中显式指定 `env` 和 `cwd`
- Server 启动时打印环境信息到 stderr（不是 stdout！）
- 使用绝对路径
### 4.4 stdout 污染导致通信失败
**坑：** Server 代码中 `print()` / `console.log()` 写到 stdout，被当作协议消息解析失败。
**正确做法：**
- 日志、调试信息一律写 stderr
- Python：`print(..., file=sys.stderr)` 或用 logging
- Node.js：`console.error()` 而非 `console.log()`
### 4.5 超时与长时间操作
**坑：** 工具执行时间过长，Client 超时断开。
**解决方案：**
- 利用 progress 通知（`notifications/progress`）上报进度
- 拆分为"提交任务"和"查询结果"两个工具
- Client 端配置合理的超时时间
- 支持 cancellation（`notifications/cancelled`）
### 4.6 错误处理不当
**坑：** Server 异常直接崩溃，或返回模糊错误信息，模型无法理解和恢复。
**正确做法：**
- 返回清晰的错误描述，告诉模型失败原因和建议
- 区分可重试错误和不可重试错误
- 不要抛未捕获异常导致 Server 进程退出
- `isError: true` 标记错误响应
```python
@server.call_tool()
async def call_tool(name, arguments):
    try:
        result = do_something(arguments)
        return [TextContent(type="text", text=result)]
    except PermissionError:
        return [TextContent(
            type="text",
            text="Permission denied. The API key may lack write access. Please check credentials."
        )], True  # isError=True
```
### 4.7 安全性问题
**坑：**
- Server 暴露了危险操作（删库、发送邮件）无需确认
- 敏感信息（API Key、Token）通过消息传递泄露给模型
- Prompt Injection 导致模型调用非预期工具
**正确做法：**
- 危险操作加入人工确认环节（Human-in-the-Loop）
- 凭证通过环境变量传入 Server，不暴露给模型
- 实施最小权限原则：只暴露必要的工具
- 输入验证：Server 端校验参数合法性
- 考虑工具调用的审计日志
### 4.8 资源和工具的混淆
**坑：** 把只读数据查询做成 Tool，把有副作用的操作做成 Resource。
**正确区分：**
- **Resource**：只读、幂等、获取上下文（文件内容、配置信息）
- **Tool**：有副作用、或需要模型推理决定参数（搜索、创建、修改）
- 经验法则：模型需要"决定做什么"→ Tool；应用需要"提供背景"→ Resource
### 4.9 状态管理
**坑：** Server 是无状态的但工具调用需要上下文（登录态、会话 ID）。
**解决方案：**
- 在 Server 端维护会话状态（适合 stdio 模式，一个 Client 对应一个进程）
- 通过参数透传状态（适合无状态 HTTP Server）
- 利用 Resource 暴露当前状态
### 4.10 版本兼容与协议升级
**坑：** MCP 协议还在快速迭代，SDK 版本间不兼容。
**建议：**
- 锁定 SDK 版本，升级前测试
- 关注 `protocolVersion` 字段，做好版本协商
- 关注 MCP spec changelog
---
## 五、MCP 设计模式与最佳实践
### 5.1 组合模式
```plain text
[文件系统 MCP Server] + [Git MCP Server] + [CI/CD MCP Server]
         ↓                     ↓                    ↓
              统一通过 MCP Client 接入 AI 应用
```
每个 Server 职责单一，通过组合提供完整能力。
### 5.2 网关模式
```plain text
MCP Client → MCP Gateway Server → 多个下游 API/服务
```
一个 MCP Server 聚合多个内部服务，统一暴露给 Client。适合企业内部多系统集成。
### 5.3 分层工具设计
```plain text
Level 1: route_request（确定用户意图分类）
Level 2: 加载对应分类的具体工具集
Level 3: 执行具体工具
```
解决工具过多问题，让模型分步决策。
### 5.4 幂等性设计
- 工具调用可能因重试被多次执行
- 写操作需要幂等设计（先检查再执行、用唯一 ID）
- 返回结果应包含操作状态，便于模型判断是否需要重试
---
## 六、MCP 生态现状（2025）
### 6.1 官方支持
- Claude Desktop / Claude Code / Claude iOS 原生支持 MCP
- Anthropic 维护官方 SDK（Python、TypeScript）
- 官方示例 Server：filesystem、GitHub、GitLab、Slack、PostgreSQL 等
### 6.2 社区生态
- Cursor、Windsurf、Cline 等 IDE 支持 MCP
- OpenAI 宣布支持 MCP
- 开源 MCP Server 仓库（mcp-servers）
- MCP Hub / Smithery 等 Server 发现平台
### 6.3 企业落地场景
- 内部知识库检索
- 数据库查询助手
- DevOps 工具链集成（CI/CD、监控、日志）
- 客服系统（工单、订单查询）
- 代码仓库操作
---
## 七、面试高频问题
1. **MCP 的本质是什么？和 Function Calling 什么关系？**
→ MCP 是应用层标准化协议，Function Calling 是底层模型能力。MCP 建立在 FC 之上，解决工具接入的 M×N 问题。
2. **MCP 的三大原语分别是什么？适用场景？**
→ Tools（模型调用操作）、Resources（应用获取数据）、Prompts（用户选择模板）。控制权不同。
3. **stdio 和 HTTP+SSE 传输方式的区别和选型？**
→ stdio 适合本地、安全、简单；HTTP+SSE 适合远程、多用户、可扩展。
4. **如何保证 MCP Server 的安全性？**
→ 最小权限、输入校验、凭证隔离、人工确认、审计日志。
5. **工具太多怎么办？**
→ 动态加载、分层路由、合并相似工具、描述优化。
6. **MCP 相比 OpenAPI/Swagger 的优势？**
→ 专为 AI 设计：支持模型自主发现和决策、双向通信、流式响应、标准化的 AI 交互模式。
7. **怎么调试 MCP Server？**
→ 使用 MCP Inspector 工具、stderr 日志、协议级抓包、单元测试工具调用。
8. **MCP 的局限性？**
→ 协议仍在演进中、生态碎片化、无统一鉴权标准、Server 质量参差不齐、性能开销（多一层 RPC）。