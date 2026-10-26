# RPC 概述
## 什么是 RPC
RPC（Remote Procedure Call，远程过程调用）是一种通信思想：让调用远程服务像调用本地函数一样简单，调用方无需关心网络传输、序列化、服务寻址等底层细节。
```python
# 理想中的 RPC 体验
result = user_service.get_user(user_id=123)  # 看起来是本地调用，实际经过了网络
```
一次 RPC 调用的核心流程：
1. 客户端调用本地代理（Stub）
2. 序列化参数为字节流
3. 通过网络发送到服务端
4. 服务端反序列化，执行真实方法
5. 将返回值序列化后发回客户端
6. 客户端反序列化，得到结果
RPC 框架通常需要解决：服务发现、序列化、网络传输、负载均衡、容错与服务治理等问题。
---
## 常见 RPC 框架
### Dubbo
Apache Dubbo 是阿里巴巴开源的高性能 Java RPC 框架，也是国内使用最广泛的 RPC 框架之一。
- **协议可插拔**：支持 Dubbo 协议、Triple（兼容 gRPC，基于 HTTP/2）、REST 等
- **注册中心可插拔**：ZooKeeper、Nacos、Consul 等
- **服务治理能力强**：内置负载均衡、熔断、限流、灰度路由
- **序列化**：Hessian2、Protobuf、JSON 等
- **Dubbo 3.x**：引入 Triple 协议，支持应用级服务发现，拥抱云原生
适用场景：Java 技术栈的内部微服务通信。
### HSF
HSF（High-Speed Service Framework）是阿里巴巴集团内部的 RPC 框架。
- **自研 TCP 协议**，基于 Netty，性能极高
- 深度集成集团内部基础设施（ConfigServer、Diamond、鹰眼链路追踪）
- 支持同机房优先、单元化路由等大规模分布式场景下的特色能力
- HSF 3.x 底层已与 Dubbo 3.x 融合
- 不开源，仅限阿里集团内部使用
适用场景：阿里集团内部微服务通信。
### gRPC
gRPC 是 Google 开源的高性能 RPC 框架。
- **基于 HTTP/2 + Protobuf**，性能优异
- **跨语言能力强**：官方支持 Go、Java、Python、C++、Rust 等十多种语言
- **支持流式调用**：单向流、双向流
- 使用 `.proto` 文件定义接口（IDL），自动生成多语言代码
- 云原生生态集成良好（Kubernetes、Istio、Envoy）
适用场景：跨语言服务通信、云原生微服务、需要流式传输的场景。
---
## Python 推荐的 RPC 框架
| 框架 | 说明 |
| --- | --- |
| **gRPC (grpcio)** | Google 官方 Python 实现，性能好，跨语言，适合生产环境 |
| **Thrift** | Apache Thrift 的 Python 实现，跨语言，IDL 代码生成 |
| **RPyC** | 纯 Python RPC 框架，使用简单，适合 Python 对 Python 的场景 |
| **Nameko** | 基于 RabbitMQ 的微服务框架，内置 RPC 支持，轻量易用 |
| **zerorpc** | 基于 ZeroMQ + MessagePack，简洁高效，适合内部工具 |
| **JSON-RPC 库** | `jsonrpcserver` / `jsonrpcclient`，轻量标准，适合简单场景 |
推荐选择：
- 生产级、跨语言 → **gRPC**
- 纯 Python、快速开发 → **RPyC** 或 **Nameko**
- 轻量、与 AI/MCP 生态对接 → **JSON-RPC**
---
## JSON-RPC
JSON-RPC 是一种轻量级的 RPC 协议，使用 JSON 作为数据格式。
### 协议规范
请求：
```json
{
  "jsonrpc": "2.0",
  "method": "get_user",
  "params": {"user_id": 123},
  "id": 1
}
```
响应：
```json
{
  "jsonrpc": "2.0",
  "result": {"name": "Alice", "age": 30},
  "id": 1
}
```
### 特点
- **极其简单**：整个规范只有一页，易于实现和调试
- **传输无关**：可以跑在 HTTP、WebSocket、TCP、stdio 等任何传输层上
- **无服务发现**：协议本身不涉及注册、发现、负载均衡，需要自行处理
- **弱类型**：没有 IDL，接口契约靠文档约定
### 与其他 RPC 框架的对比
|  | JSON-RPC | gRPC | Dubbo |
| --- | --- | --- | --- |
| 数据格式 | JSON（文本） | Protobuf（二进制） | Hessian2（二进制） |
| 性能 | 一般 | 高 | 高 |
| 复杂度 | 极低 | 中 | 高 |
| 跨语言 | 天然支持 | 需要代码生成 | 主要 Java |
| 服务治理 | 无 | 基础 | 丰富 |
适用场景：轻量级服务通信、工具集成、AI 应用协议的底层承载（如 MCP）。
---
## MCP（Model Context Protocol）
MCP 是 Anthropic 提出的开放协议，定义了 AI 模型（Agent）与外部工具/数据源之间的标准通信方式。可以理解为 **AI 时代的 RPC 协议**。
### 核心思想
传统 RPC 让**程序**调用远程服务；MCP 让 **AI 模型**调用外部工具。
```plain text
传统 RPC：  Service A  ──调用──→  Service B
MCP：       AI Agent   ──调用──→  Tool / 数据源
```
### 架构
```plain text
AI 模型 (Client)
    │
    ├── MCP Client
    │       │
    │       ▼
    │   MCP Server A ── 文件系统、数据库、搜索引擎 ...
    │       │
    │       ▼
    │   MCP Server B ── 内部 API、第三方服务 ...
    │
    └── 模型根据 Tool 描述自主决定调用哪个工具、传什么参数
```
### 协议基础
MCP 底层基于 **JSON-RPC 2.0**，传输方式支持：
- **HTTP + SSE**（Server-Sent Events）：适用于远程服务
- **stdio**：适用于本地进程通信
### 核心概念
| 概念 | 说明 |
| --- | --- |
| **Tool** | 模型可调用的工具，包含名称、描述、参数 JSON Schema |
| **Resource** | 模型可读取的数据源（文件、数据库等） |
| **Prompt** | 预定义的提示词模板 |
### 与传统 RPC 的关键差异
|  | 传统 RPC | MCP |
| --- | --- | --- |
| 调用方 | 程序（确定性逻辑） | AI 模型（概率推理） |
| 接口描述 | IDL / 强类型接口 | JSON Schema + 自然语言描述 |
| 调用决策 | 程序员硬编码 | 模型自主判断何时调用 |
| 发现机制 | 注册中心 | Tool 列表 + 语义描述 |
MCP 的 Tool 需要清晰的自然语言 description，因为"调用方"是 AI 模型——它需要理解工具的用途才能正确使用。
### Python 中使用 MCP
```python
# 使用 mcp 官方 SDK 创建一个 MCP Server
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("my-tools")

@mcp.tool()
def get_weather(city: str) -> str:
    """获取指定城市的天气信息"""
    return f"{city}：晴，25°C"

mcp.run()
```
---
## 总结
```plain text
RPC（思想）
 ├── 传统 RPC 框架（服务间通信）
 │     ├── Dubbo（Java 开源，国内主流）
 │     ├── HSF（阿里内部，已融合 Dubbo）
 │     ├── gRPC（Google，跨语言，云原生）
 │     └── Thrift（Facebook，逐渐式微）
 │
 ├── 轻量协议
 │     └── JSON-RPC（极简，传输无关）
 │
 └── AI 时代的 RPC
       └── MCP（基于 JSON-RPC，面向 AI Agent 调用工具）
```
核心关系：RPC 是目标，Dubbo/HSF/gRPC 是面向服务的实现，JSON-RPC 是轻量协议规范，MCP 是基于 JSON-RPC 面向 AI 场景的上层应用协议。