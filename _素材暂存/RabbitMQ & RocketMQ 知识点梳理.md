# RabbitMQ & RocketMQ 知识点梳理
## 一、消息队列基础
**为什么需要消息队列？**
- **解耦**：上下游系统独立演进，互不依赖
- **异步**：非核心流程异步处理，提升响应速度
- **削峰填谷**：应对流量洪峰，保护下游系统
- **数据分发**：一份数据多个消费者各自处理
**消息队列的核心问题**
- 如何保证消息不丢失？
- 如何保证消息不重复消费？
- 如何保证消息有序？
- 如何处理消息堆积？
---
## 二、RabbitMQ
### 2.1 核心概念
| 概念 | 说明 |
| --- | --- |
| Producer | 消息生产者 |
| Consumer | 消息消费者 |
| Broker | 消息服务器 |
| VHost | 虚拟主机，逻辑隔离 |
| Exchange | 交换机，路由消息 |
| Queue | 队列，存储消息 |
| Binding | Exchange 与 Queue 的绑定关系 |
| Routing Key | 路由键，决定消息去向 |
| Channel | 信道，复用 TCP 连接 |
### 2.2 Exchange 类型
| 类型 | 路由规则 | 适用场景 |
| --- | --- | --- |
| Direct | 精确匹配 Routing Key | 点对点，指定队列 |
| Fanout | 广播到所有绑定队列 | 广播通知 |
| Topic | 通配符匹配（`*` 一个词，`#` 多个词） | 灵活的发布订阅 |
| Headers | 匹配消息头属性 | 少用，性能差 |
### 2.3 工作模式
- **Simple**：一个生产者 → 一个队列 → 一个消费者
- **Work Queue**：多个消费者竞争消费同一队列（负载均衡）
- **Publish/Subscribe**：Fanout Exchange 广播
- **Routing**：Direct Exchange 定向路由
- **Topics**：Topic Exchange 模式匹配
- **RPC**：请求-响应模式，利用 reply_to 和 correlation_id
### 2.4 消息可靠性
**生产端可靠投递**
- **事务机制**：`txSelect` / `txCommit` / `txRollback`，性能差
- **Confirm 模式**（推荐）：异步确认，Broker 收到后回调 ack/nack
- **Mandatory 参数**：消息无法路由时返回给生产者
**Broker 持久化**
- Exchange 持久化：声明时设置 `durable=true`
- Queue 持久化：声明时设置 `durable=true`
- Message 持久化：发送时设置 `deliveryMode=2`
- 注意：持久化 ≠ 100% 不丢，需配合镜像队列
**消费端可靠消费**
- 手动 ACK（关闭 autoAck）
- `basicAck`：确认消费成功
- `basicNack` / `basicReject`：拒绝并可选择重新入队
- 预取数量 `prefetchCount`：控制消费速率
### 2.5 高可用架构
**普通集群模式**
- 队列数据只存在于声明它的节点
- 其他节点存储元数据 + 指针，转发请求
- 不具备高可用，节点挂了队列就不可用
**镜像队列模式（Classic Mirrored Queue）**
- 队列数据在多个节点间同步复制
- 策略配置：`ha-mode`（all / exactly / nodes）
- 主节点挂了，从节点自动提升
- 缺点：性能开销大，同步有延迟
**Quorum Queue（3.8+，推荐）**
- 基于 Raft 协议的高可用队列
- 强一致性，多数节点确认才返回
- 自动选举 Leader，故障自动恢复
- 替代镜像队列的方案
### 2.6 死信队列（DLX）
消息变为死信的条件：
- 消息被 `basicNack` / `basicReject` 且 `requeue=false`
- 消息 TTL 过期
- 队列达到最大长度
应用：延迟队列（TTL + DLX）、异常消息兜底
### 2.7 延迟队列实现
- **TTL + DLX**：设置消息/队列 TTL，过期后转入死信队列消费
	- 问题：队列级 TTL 按顺序过期，消息级 TTL 可能阻塞
- **延迟插件**（rabbitmq_delayed_message_exchange）：推荐方案
- **应用场景**：订单超时取消、延迟通知、定时任务
### 2.8 性能与调优
- 单机吞吐量：万级 \~ 十万级 QPS
- 消息堆积能力有限（内存为主）
- 调优手段：
	- 增加 prefetchCount
	- 多队列 + 多消费者并行
	- 惰性队列（Lazy Queue）：消息直接写磁盘
	- 批量确认
---
## 三、RocketMQ
### 3.1 核心概念
| 概念 | 说明 |
| --- | --- |
| Producer | 消息生产者 |
| Consumer | 消息消费者 |
| NameServer | 注册中心，无状态，存储路由信息 |
| Broker | 消息存储与转发服务器 |
| Topic | 消息主题，逻辑分类 |
| MessageQueue | Topic 下的分区（类似 Kafka Partition） |
| Tag | 消息标签，二级分类过滤 |
| ConsumerGroup | 消费者组，组内负载均衡 |
| ProducerGroup | 生产者组，用于事务消息回查 |
| Offset | 消费位点 |
### 3.2 架构设计
```plain text
Producer → NameServer（获取路由）→ Broker（Master/Slave）
Consumer → NameServer（获取路由）→ Broker（拉取消息）
```
**NameServer**
- 无状态，各节点独立，不互相通信
- Broker 定时（30s）向所有 NameServer 注册心跳
- 客户端启动时随机连一个 NameServer 获取路由
- 比 ZooKeeper 更轻量，适合 MQ 场景
**Broker**
- Master-Slave 架构（同步/异步复制）
- CommitLog：所有消息顺序写入一个文件（追加写，性能极高）
- ConsumeQueue：按 Topic + QueueId 建索引（定长 20 字节）
- IndexFile：按 Message Key 建哈希索引
### 3.3 消息类型
| 类型 | 说明 | 场景 |
| --- | --- | --- |
| 普通消息 | 无特殊保证 | 通用业务 |
| 顺序消息 | 同一 Queue 内严格有序 | 订单状态流转 |
| 延迟消息 | 指定延迟级别后投递 | 超时处理、定时任务 |
| 事务消息 | 半消息 + 本地事务 + 回查 | 分布式事务最终一致性 |
| 批量消息 | 多条消息一次发送 | 提高吞吐 |
### 3.4 顺序消息
**全局有序**
- Topic 只设一个 MessageQueue
- 牺牲并行度，吞吐很低
**分区有序**（常用）
- 同一业务 ID（如订单号）的消息发到同一 Queue
- MessageQueueSelector 自定义路由策略
- 消费端单线程消费同一 Queue
### 3.5 事务消息
```plain text
1. Producer 发送半消息（Half Message）→ Broker 存储但不投递
2. Producer 执行本地事务
3. 根据本地事务结果：
   - 成功 → Commit（消息投递给消费者）
   - 失败 → Rollback（删除半消息）
4. 若超时未确认 → Broker 回查 Producer 本地事务状态
```
- 解决分布式事务最终一致性问题
- 回查次数和间隔可配置（默认最多 15 次）
- ProducerGroup 的意义：回查时可找到同组任意 Producer
### 3.6 消息可靠性
**生产端**
- 同步发送（SendResult 确认）
- 重试机制：默认重试 2 次，自动切换 Broker
- 事务消息保证本地事务与消息一致
**Broker 端**
- 同步刷盘（SYNC_FLUSH）：性能低但可靠
- 异步刷盘（ASYNC_FLUSH）：性能高但可能丢少量消息
- 同步复制（SYNC_MASTER）：Master-Slave 同步
- 异步复制（ASYNC_MASTER）：性能高但 Master 宕机可能丢数据
**消费端**
- 消费失败返回 `RECONSUME_LATER`，进入重试队列
- 重试次数递增延迟：10s、30s、1min、2min... 最多 16 次
- 超过重试次数进入死信队列（%DLQ%ConsumerGroup）
- Offset 提交：广播模式本地存储，集群模式 Broker 存储
### 3.7 消费模式
**集群模式（Clustering）**
- 同一 ConsumerGroup 内负载均衡，一条消息只被一个实例消费
- 支持 Rebalance（增减消费者自动重分配）
**广播模式（Broadcasting）**
- 同一 ConsumerGroup 内每个实例都收到全量消息
- Offset 本地存储，不支持重试
**Pull vs Push**
- Push 模式：本质是长轮询 Pull（挂起请求等待新消息）
- Pull 模式：消费者主动拉取，可控但实时性低
- 4.x 引入 Pop 模式：服务端分配，避免 Rebalance 问题
### 3.8 高可用架构
**Master-Slave（4.x）**
- BrokerId=0 为 Master，其他为 Slave
- 同步复制 + 同步刷盘 = 最高可靠性
- Master 挂了 Slave 可读但不可写（需手动切换）
**Dledger 模式（4.5+）**
- 基于 Raft 协议自动选主
- Master 宕机自动切换，无需人工介入
- 至少 3 节点
**Controller 模式（5.0）**
- 独立 Controller 组件管理选主
- 支持混合部署，更灵活
### 3.9 消息堆积处理
- RocketMQ 天然支持大量堆积（CommitLog 顺序写磁盘）
- 堆积百亿级消息性能不明显下降
- 处理方案：
	- 增加消费者实例（不超过 Queue 数量）
	- 临时扩容 Queue + 转发
	- 跳过非重要消息
	- 提升单条消费速度
### 3.10 存储设计（高频考点）
**CommitLog**
- 所有 Topic 消息混合顺序写入
- 单文件默认 1GB，写满创建新文件
- 顺序写 + PageCache + 零拷贝（mmap）= 极高写入性能
**ConsumeQueue**
- 每个 Topic 的每个 Queue 对应一个文件
- 定长记录（8B offset + 4B size + 8B tag hashcode = 20B）
- 类比数据库的二级索引
**IndexFile**
- 按 Message Key 建哈希索引
- 支持按 Key 或时间范围查询消息
**零拷贝**
- mmap（内存映射）：适合小文件读写（ConsumeQueue）
- sendfile：适合大文件传输（CommitLog 消费读取）
### 3.11 延迟消息
**4.x 版本**
- 固定延迟级别：1s、5s、10s、30s、1m、2m、... 2h（18 个级别）
- 实现：延迟消息先写入 SCHEDULE_TOPIC_XXXX，定时任务扫描投递
**5.x 版本**
- 支持任意时间精度的定时消息
- 基于时间轮实现
---
## 四、RabbitMQ vs RocketMQ 对比
| 维度 | RabbitMQ | RocketMQ |
| --- | --- | --- |
| 开发语言 | Erlang | Java |
| 协议 | AMQP、MQTT、STOMP | 自定义协议 |
| 单机吞吐 | 万级 | 十万级 |
| 消息堆积 | 较弱（内存为主） | 极强（磁盘顺序写） |
| 延迟 | 微秒级 | 毫秒级 |
| 顺序消息 | 单队列保证 | 分区有序 |
| 事务消息 | 不支持（仅有 TX 事务） | 原生支持半消息事务 |
| 延迟消息 | 插件/TTL+DLX | 原生支持 |
| 消息过滤 | Routing Key + Header | Tag + SQL92 表达式 |
| 管理界面 | 自带 Web UI（功能完善） | Dashboard（5.x 改善） |
| 社区生态 | 国际社区活跃 | 阿里开源，国内生态强 |
| 适用场景 | 中小规模、路由灵活、协议丰富 | 大规模、高吞吐、金融级可靠 |
---
## 五、选型建议
| 场景 | 推荐 | 理由 |
| --- | --- | --- |
| 中小项目、团队偏好 Erlang 运维 | RabbitMQ | 轻量、易用、功能完备 |
| 大流量、消息堆积严重 | RocketMQ | 磁盘存储、堆积无压力 |
| 需要事务消息 | RocketMQ | 原生半消息事务 |
| 需要复杂路由逻辑 | RabbitMQ | Exchange 类型丰富 |
| 金融级可靠性 | RocketMQ | 同步刷盘 + 同步复制 |
| 多语言客户端 | RabbitMQ | AMQP 协议，客户端丰富 |
| 日志/大数据场景 | Kafka | 吞吐最高，生态最好 |
---
## 六、高频面试题
### 通用问题
1. **如何保证消息不丢失？**
	- 生产端：确认机制（Confirm / SendResult）
	- Broker：持久化 + 副本（镜像队列 / 同步复制）
	- 消费端：手动 ACK，消费成功后再确认
2. **如何保证消息不重复消费（幂等性）？**
	- 全局唯一 MessageId + 去重表
	- 业务层幂等：数据库唯一键、乐观锁、状态机
	- Redis Set 去重（适合时间窗口内去重）
3. **如何保证消息有序？**
	- 单 Queue/Partition + 单消费者
	- 按业务键哈希到同一分区
	- 消费端单线程或内存队列排序
4. **消息堆积了怎么办？**
	- 紧急：增加消费者、扩容队列
	- 临时：跳过不重要消息、转发到新 Topic
	- 根本：优化消费逻辑、提升处理速度
5. **如何实现分布式事务？**
	- RocketMQ 事务消息（半消息 + 回查）
	- 本地消息表 + 定时补偿
	- Saga 模式
### RabbitMQ 专项
1. **Confirm 模式和事务模式的区别？** → Confirm 异步高性能，事务同步低性能
2. **镜像队列和 Quorum Queue 的区别？** → Quorum 基于 Raft 强一致，推荐替代镜像队列
3. **Channel 和 Connection 的关系？** → Connection 是 TCP 连接，Channel 是逻辑复用
### RocketMQ 专项
1. **为什么 CommitLog 设计为混合存储？** → 顺序写性能最优，通过 ConsumeQueue 做逻辑分发
2. **NameServer 为什么不用 ZooKeeper？** → 更轻量、无状态、无选举开销，够用即可
3. **消费者 Rebalance 的触发时机？** → 消费者上下线、Queue 数量变化、定时 20s 触发
4. **事务消息回查失败怎么办？** → 默认最多回查 15 次，超过后回滚