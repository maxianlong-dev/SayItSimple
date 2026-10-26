# OpenTelemetry 入门指南
## 一、为什么需要 OpenTelemetry？
### 问题场景
想象你运营一个电商网站，某天用户投诉："下单好慢！"
你面对的是这样一个系统：
```plain text
用户 → 网关 → 订单服务 → 库存服务 → 支付服务 → 数据库
```
**问题来了：**
- 慢在哪一步？网关？订单？支付？数据库？
- 是所有请求都慢，还是某些用户慢？
- 是今天才慢，还是一直在恶化？
如果没有可观测性工具，你只能靠猜、靠翻日志、靠祈祷。
### 传统方案的痛点
| 痛点 | 说明 |
| --- | --- |
| 日志散落各处 | 每个服务各打各的日志，无法串联一次完整请求 |
| 指标口径不统一 | A 服务用 Prometheus，B 服务用 StatsD，对不上 |
| 厂商锁定 | 用了某家 APM（如 DataDog），想换就得改所有代码 |
| 缺乏标准 | 链路追踪用 Zipkin 格式？Jaeger 格式？各搞各的 |
### OpenTelemetry 的定位
> **一句话：OpenTelemetry 是可观测性领域的"USB-C 接口"——统一标准，随意插拔后端。**
它不是一个监控平台，而是一套**采集标准 + SDK**。你用它采集数据，想发给谁就发给谁。
---
## 二、核心概念与术语
### 可观测性三大支柱
| 支柱 | 比喻 | 回答的问题 |
| --- | --- | --- |
| **Traces（链路）** | 快递物流轨迹 | 这次请求经过了哪些服务？每步多久？ |
| **Metrics（指标）** | 汽车仪表盘 | 系统整体健不健康？每秒多少请求？错误率多少？ |
| **Logs（日志）** | 黑匣子录音 | 某个时刻具体发生了什么事？ |
### 详细术语表
#### Trace（链路）
**比喻：一张快递运单**
从发件到签收的完整记录。在软件中，就是一个用户请求从进入系统到返回响应的完整路径。
```plain text
Trace ID: abc-123（全局唯一，贯穿所有服务）

用户点击"下单"
├── 网关接收 (2ms)
├── 订单服务处理 (80ms)
│   ├── 校验库存 (20ms)
│   └── 写入数据库 (50ms)  ← 瓶颈！
└── 返回响应 (1ms)
```
#### Span（跨度）
**比喻：快递单上的每一个中转记录**
一个 Trace 由多个 Span 组成，每个 Span 代表一个操作单元。
Span 包含的信息：
| 字段 | 说明 | 例子 |
| --- | --- | --- |
| Name | 操作名 | "查询用户表" |
| Start/End Time | 起止时间 | 耗时 30ms |
| Status | 成功/失败 | ERROR |
| Attributes | 附加信息 | `user.id=123`, `db.statement="SELECT..."` |
| Parent Span ID | 父级 Span | 表示"谁调用了我" |
#### Context Propagation（上下文传播）
**比喻：接力赛中传递接力棒**
服务 A 调用服务 B 时，通过 HTTP Header 传递 Trace ID：
```plain text
服务A                          服务B
  │                              │
  │── HTTP Request ─────────────→│
  │   Header:                    │
  │   traceparent: 00-abc123-... │
  │                              │
  │   "我是链路abc123的一部分，   │
  │    继续用这个ID记录"         │
```
#### Instrumentation（插桩）
**比喻：在公路上安装摄像头**
不改变车辆行为，只是在关键位置添加观测点。
两种方式：
| 方式 | 说明 | 适合场景 |
| --- | --- | --- |
| 自动插桩 | 零代码，自动拦截已知框架 | 快速接入，覆盖通用场景 |
| 手动插桩 | 写代码创建自定义 Span | 业务逻辑中的关键路径 |
#### Exporter（导出器）
**比喻：摄像头拍到的视频存到哪个硬盘**
OpenTelemetry 采集数据后，通过 Exporter 发送到你选择的后端：
```plain text
你的应用 (SDK)
    │
    ├──→ Jaeger（链路可视化）
    ├──→ Prometheus（指标监控）
    ├──→ Elasticsearch（日志搜索）
    └──→ 阿里云 ARMS / SLS
```
#### Collector（收集器）
**比喻：快递分拣中心**
独立部署的中间层服务，负责接收、处理、转发数据：
```plain text
服务A ─┐                     ┌──→ Jaeger
服务B ─┼──→ OTel Collector ──┼──→ Prometheus
服务C ─┘    (分拣中心)        └──→ 云服务
```
好处：应用只需发给 Collector，换后端时只改 Collector 配置，不动业务代码。
#### Resource（资源）
**比喻：快递上的发件人信息**
描述"谁产生了这条数据"：
```python
Resource(attributes={
    "service.name": "order-service",    # 哪个服务
    "service.version": "1.2.0",         # 什么版本
    "deployment.environment": "prod",   # 什么环境
})
```
#### Semantic Conventions（语义约定）
**比喻：所有快递公司统一使用相同的面单格式**
规定了通用的属性命名，比如：
- HTTP 请求用 `http.request.method`、`http.response.status_code`
- 数据库用 `db.system`、`db.statement`
- 消息队列用 `messaging.system`、`messaging.destination`
这样不管是什么语言、什么框架，产生的数据格式一致，后端可以统一解析。
### Metrics 的三种类型
| 类型 | 比喻 | 例子 |
| --- | --- | --- |
| **Counter（计数器）** | 汽车里程表，只增不减 | 总请求数、总错误数 |
| **Gauge（仪表）** | 温度计，可升可降 | 当前内存使用、当前连接数 |
| **Histogram（直方图）** | 考试分数分布 | 请求延迟分布（P50/P95/P99） |
---
## 三、OpenTelemetry 的架构与实现
### 整体架构
```plain text
┌─────────────────────────────────────────────────────┐
│                    你的应用                           │
│                                                     │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐      │
│  │  OTel API │  │  OTel SDK │  │ 自动插桩   │      │
│  └───────────┘  └───────────┘  └───────────┘      │
│        │              │              │              │
│        └──────────────┼──────────────┘              │
│                       ▼                             │
│              ┌─────────────────┐                    │
│              │    Exporter     │                    │
│              └────────┬────────┘                    │
└───────────────────────┼─────────────────────────────┘
                        ▼
              ┌─────────────────┐
              │  OTel Collector │  （可选，推荐生产用）
              └────────┬────────┘
                       ▼
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
   ┌────────┐    ┌──────────┐   ┌────────┐
   │ Jaeger │    │Prometheus│   │  SLS   │
   └────────┘    └──────────┘   └────────┘
```
### API vs SDK
| 层级 | 职责 | 比喻 |
| --- | --- | --- |
| **API** | 定义接口（创建 Span、记录 Metric） | 插座标准 |
| **SDK** | 具体实现（采样、批量发送、资源管理） | 插座背后的电路 |
分离的好处：库作者只依赖轻量的 API，不引入 SDK 的重量；最终用户在应用层配置 SDK。
---
## 四、常见的可观测性框架对比
| 框架 | 类型 | 优势 | 劣势 |
| --- | --- | --- | --- |
| **OpenTelemetry** | 开源标准 | 厂商中立、社区大、一套覆盖三大支柱 | 需要自己搭后端 |
| **Jaeger** | 链路追踪 | 成熟、已原生支持 OTel | 只做 Traces |
| **Zipkin** | 链路追踪 | 轻量、易部署 | 功能较简单 |
| **Prometheus** | 指标监控 | 事实标准、生态丰富 | 只做 Metrics |
| **ELK Stack** | 日志分析 | 强大的搜索能力 | 资源消耗大 |
| **DataDog** | 商业 APM | 开箱即用、全家桶 | 贵、厂商锁定 |
| **SkyWalking** | 开源 APM | 国内社区活跃、功能全 | 与 OTel 标准有差异 |
### OpenTelemetry 与它们的关系
```plain text
OpenTelemetry 不是替代品，而是"统一采集层"：

  你的代码 → OpenTelemetry → Jaeger      (看链路)
                           → Prometheus  (看指标)
                           → ELK         (看日志)
                           → DataDog     (全部)
```
---
## 五、OpenTelemetry 的优势
### 1. 厂商中立，不被锁定
```plain text
以前：代码里写死 DataDog SDK → 换监控 = 改所有代码
现在：代码里用 OTel SDK → 换监控 = 改一行 Exporter 配置
```
### 2. 统一标准
一套 API 覆盖 Traces + Metrics + Logs，不用学三套工具。
### 3. 语言覆盖广
官方支持 C++、Java、Python、Go、JS、.NET、PHP、Ruby、Rust、Swift 等 13 种语言，API 用法一致。
### 4. 自动插桩 + 零代码方案
主流框架（FastAPI、Spring Boot、Express.js、gRPC 等）都有现成的自动插桩库，几行配置就能出数据。
### 5. 云原生友好
- Kubernetes Operator 自动注入
- 与 Service Mesh（Istio）深度集成
- CNCF 毕业项目，云厂商全面支持
### 6. 社区活跃
CNCF 中活跃度仅次于 Kubernetes 的项目，400+ 贡献者持续迭代。
---
## 六、快速上手示例（Python + FastAPI）
### 安装
```bash
pip install opentelemetry-api \
            opentelemetry-sdk \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-fastapi
```
### 最小 Demo
```python
from fastapi import FastAPI
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import ConsoleSpanExporter, BatchSpanProcessor
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor

# --- 配置 OpenTelemetry ---
provider = TracerProvider()
# 把数据打印到控制台（生产环境换成 OTLPSpanExporter）
provider.add_span_processor(BatchSpanProcessor(ConsoleSpanExporter()))
trace.set_tracer_provider(provider)

# --- 创建 FastAPI 应用 ---
app = FastAPI()

# --- 一行代码自动插桩 ---
FastAPIInstrumentor.instrument_app(app)

# --- 手动插桩示例 ---
tracer = trace.get_tracer(__name__)

@app.get("/order/{order_id}")
async def get_order(order_id: int):
    # 自定义一个 Span
    with tracer.start_as_current_span("查询订单") as span:
        span.set_attribute("order.id", order_id)
        # 模拟业务逻辑...
        return {"order_id": order_id, "status": "已发货"}
```
### 运行
```bash
uvicorn main:app --reload
# 访问 http://localhost:8000/order/42
# 控制台会输出 Span 信息
```
---
## 七、概念总结速查表
| 术语 | 一句话解释 | 比喻 |
| --- | --- | --- |
| Trace | 一次请求的完整调用链 | 快递运单 |
| Span | 调用链中的一个操作 | 运单上的一个中转站 |
| Context Propagation | 跨服务传递追踪信息 | 接力赛传递接力棒 |
| Instrumentation | 在代码中埋点采集数据 | 在路上装摄像头 |
| Exporter | 数据发到哪个后端 | 视频存到哪个硬盘 |
| Collector | 统一接收转发的中间层 | 快递分拣中心 |
| Resource | 标识数据来源的元信息 | 发件人信息 |
| Semantic Conventions | 属性命名标准 | 统一面单格式 |
| Sampling | 只采集部分数据降低开销 | 抽样检查 |
| Baggage | 跨服务传递业务数据 | 快递里夹带的小纸条 |
---
## 八、学习路径建议
```plain text
第1步：跑通 Demo
       → 本地启动 Jaeger + FastAPI，看到链路图

第2步：理解数据模型
       → 搞清楚 Trace / Span / Attribute 的关系

第3步：学习自动插桩
       → 用零代码方案覆盖 HTTP、DB、Redis 等

第4步：加入 Metrics
       → 用 Prometheus + Grafana 做仪表盘

第5步：生产化部署
       → 部署 Collector、配置采样策略、接入云服务
```