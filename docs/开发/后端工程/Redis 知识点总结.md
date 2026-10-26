# Redis 知识点总结
## 一、Redis 简介
Redis（Remote Dictionary Server）是一个开源的、基于内存的高性能键值数据库，支持多种数据结构，常用于缓存、消息队列、分布式锁等场景。
**核心特点：**
- 基于内存，读写速度极快（10w+ QPS）
- 单线程模型（6.0 后 I/O 多线程，命令执行仍单线程）
- 支持数据持久化（RDB / AOF）
- 支持丰富的数据结构
- 支持主从复制、哨兵、集群模式
- 原子性操作
---
## 二、数据结构
### 1. String（字符串）
最基本的类型，可存储字符串、整数、浮点数，最大 512MB。
```bash
SET key value [EX seconds] [NX|XX]
GET key
INCR key          # 原子自增
INCRBY key n      # 原子增加 n
MSET k1 v1 k2 v2 # 批量设置
MGET k1 k2       # 批量获取
SETNX key value   # 不存在时才设置（分布式锁基础）
```
**应用场景：** 缓存、计数器、分布式锁、Session 共享
### 2. Hash（哈希）
键值对集合，适合存储对象。
```bash
HSET key field value
HGET key field
HMSET key f1 v1 f2 v2
HGETALL key
HINCRBY key field n
HDEL key field
```
**应用场景：** 用户信息、商品详情等对象存储
### 3. List（列表）
双向链表，支持头尾操作。
```bash
LPUSH key v1 v2   # 左侧插入
RPUSH key v1 v2   # 右侧插入
LPOP key          # 左侧弹出
RPOP key          # 右侧弹出
LRANGE key 0 -1   # 获取全部
BLPOP key timeout  # 阻塞弹出
LLEN key          # 获取长度
```
**应用场景：** 消息队列、最新消息列表、时间线
### 4. Set（集合）
无序、不重复的字符串集合。
```bash
SADD key v1 v2
SMEMBERS key      # 所有成员
SISMEMBER key v   # 是否存在
SINTER k1 k2     # 交集
SUNION k1 k2     # 并集
SDIFF k1 k2      # 差集
SRANDMEMBER key n # 随机获取
```
**应用场景：** 标签系统、共同好友、抽奖、去重
### 5. Sorted Set（有序集合）
带分数的有序集合，按 score 排序。
```bash
ZADD key score member
ZRANGE key 0 -1 [WITHSCORES]   # 正序
ZREVRANGE key 0 -1             # 倒序
ZRANK key member               # 排名
ZSCORE key member              # 分数
ZRANGEBYSCORE key min max      # 按分数范围
ZINCRBY key increment member   # 增加分数
```
**应用场景：** 排行榜、延时队列、带权重的消息
### 6. 其他数据结构
| 类型 | 说明 | 典型应用 |
| --- | --- | --- |
| Bitmap | 位图操作 | 签到、用户在线状态 |
| HyperLogLog | 基数统计（有误差） | UV 统计 |
| GEO | 地理位置 | 附近的人、距离计算 |
| Stream | 消息流（5.0+） | 消息队列（替代 List） |
---
## 三、持久化
### 1. RDB（Redis Database）
将内存数据快照写入磁盘（二进制文件 dump.rdb）。
```bash
# redis.conf 配置
save 900 1      # 900 秒内至少 1 次修改则触发
save 300 10     # 300 秒内至少 10 次修改
save 60 10000   # 60 秒内至少 10000 次修改
```
**优点：** 文件紧凑、恢复速度快、适合备份
**缺点：** 可能丢失最后一次快照后的数据
### 2. AOF（Append Only File）
以日志形式记录每个写操作。
```bash
appendonly yes
appendfsync always    # 每次写都同步（最安全，性能差）
appendfsync everysec  # 每秒同步（推荐，最多丢 1 秒数据）
appendfsync no        # 由 OS 决定
```
**AOF 重写：** 定期压缩 AOF 文件，去除冗余命令。
### 3. 混合持久化（4.0+）
RDB + AOF 结合：AOF 重写时先写 RDB 格式，再追加增量 AOF。兼顾恢复速度和数据安全。
```bash
aof-use-rdb-preamble yes
```
---
## 四、内存管理与淘汰策略
### 过期删除策略
- **惰性删除：** 访问 key 时检查是否过期
- **定期删除：** 定时随机抽取部分 key 检查
### 内存淘汰策略（maxmemory-policy）
| 策略 | 说明 |
| --- | --- |
| noeviction | 不淘汰，写入报错（默认） |
| allkeys-lru | 所有 key 中淘汰最近最少使用 |
| allkeys-lfu | 所有 key 中淘汰最不常使用（4.0+） |
| allkeys-random | 所有 key 随机淘汰 |
| volatile-lru | 有过期时间的 key 中 LRU |
| volatile-lfu | 有过期时间的 key 中 LFU |
| volatile-random | 有过期时间的 key 随机淘汰 |
| volatile-ttl | 淘汰 TTL 最短的 key |
---
## 五、高可用架构
### 1. 主从复制
```plain text
Master (读写) --> Slave1 (只读)
              --> Slave2 (只读)
```
- 全量同步：首次连接，Master 生成 RDB 发送给 Slave
- 增量同步：通过 replication offset 和 backlog 实现断点续传
- 读写分离：写走 Master，读走 Slave
### 2. 哨兵模式（Sentinel）
自动监控、故障转移。
```plain text
Sentinel1  Sentinel2  Sentinel3
    \          |          /
     Master --> Slave1, Slave2
```
**功能：**
- 监控 Master/Slave 是否正常
- 自动故障转移（Master 宕机时选举新 Master）
- 通知客户端新 Master 地址
**故障判定：**
- 主观下线（SDOWN）：单个 Sentinel 认为下线
- 客观下线（ODOWN）：多数 Sentinel 达成共识
### 3. Cluster 集群模式
去中心化分片，16384 个 hash slot 分布在多个节点。
```plain text
Node1 [0-5460]  Node2 [5461-10922]  Node3 [10923-16383]
  |                |                   |
Slave1           Slave2              Slave3
```
**特点：**
- 数据自动分片（CRC16(key) % 16384）
- 每个节点互相通信（Gossip 协议）
- 支持在线扩缩容（slot 迁移）
- 客户端重定向（MOVED / ASK）
---
## 六、常见应用场景
### 1. 分布式锁
```bash
# 加锁
SET lock_key unique_value NX EX 30

# 解锁（Lua 脚本保证原子性）
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```
**Redlock 算法：** 多节点场景下，向 N 个独立 Redis 实例加锁，多数成功才算加锁成功。
### 2. 缓存设计
**缓存穿透：** 查询不存在的数据
- 布隆过滤器拦截
- 空值缓存（短 TTL）
**缓存击穿：** 热点 key 过期瞬间大量请求
- 互斥锁（SETNX）
- 逻辑过期（不设 TTL，后台异步更新）
**缓存雪崩：** 大量 key 同时过期
- 过期时间加随机值
- 多级缓存（本地 + Redis）
- 熔断限流
### 3. 消息队列
| 方案 | 实现 | 特点 |
| --- | --- | --- |
| List | LPUSH + BRPOP | 简单，不支持消费组 |
| Pub/Sub | PUBLISH + SUBSCRIBE | 广播，消息不持久化 |
| Stream | XADD + XREADGROUP | 持久化、消费组、ACK（推荐） |
### 4. 延时队列
利用 Sorted Set，score 为执行时间戳：
```bash
ZADD delay_queue timestamp task_data
ZRANGEBYSCORE delay_queue 0 current_time LIMIT 0 1
```
### 5. 限流
滑动窗口限流（Sorted Set）：
```bash
ZADD rate_limit:user_id now now
ZREMRANGEBYSCORE rate_limit:user_id 0 (now - window)
ZCARD rate_limit:user_id  # 判断是否超限
```
---
## 七、性能优化
### 1. Pipeline（管道）
批量发送命令，减少网络 RTT：
```python
pipe = redis.pipeline()
for i in range(1000):
    pipe.set(f"key:{i}", i)
pipe.execute()
```
### 2. Lua 脚本
保证多条命令原子执行：
```bash
EVAL "redis.call('set', KEYS[1], ARGV[1]); redis.call('expire', KEYS[1], ARGV[2])" 1 mykey myval 60
```
### 3. 大 Key 问题
**识别：**
```bash
redis-cli --bigkeys
MEMORY USAGE key
```
**解决：**
- String \> 10KB：压缩或拆分
- Hash/List/Set \> 5000 元素：分片（hash tag 拆分）
- 删除大 key：UNLINK（异步删除，4.0+）
### 4. 热 Key 问题
**解决方案：**
- 本地缓存（L1 Cache）
- 读副本（多从节点分担）
- Key 拆分（key_1, key_2 随机读）
### 5. 慢查询
```bash
slowlog-log-slower-than 10000  # 超过 10ms 记录
slowlog-max-len 128
SLOWLOG GET 10                 # 查看慢查询
```
**避免：** KEYS \*、大范围 HGETALL、未加 LIMIT 的 ZRANGEBYSCORE
---
## 八、事务与 Lua
### 事务
```bash
MULTI        # 开始事务
SET k1 v1
SET k2 v2
EXEC         # 执行
# 或 DISCARD 取消
```
**注意：** Redis 事务不支持回滚，命令错误时其他命令仍执行。
### WATCH（乐观锁）
```bash
WATCH key
val = GET key
MULTI
SET key (val + 1)
EXEC  # 如果 key 被其他客户端修改，EXEC 返回 nil
```
---
## 九、Redis 6.0+ 新特性
| 版本 | 特性 |
| --- | --- |
| 6.0 | I/O 多线程、ACL 权限控制、客户端缓存 |
| 6.2 | GETDEL、COPY、LMPOP 等新命令 |
| 7.0 | Function（替代 Lua EVAL）、Sharded Pub/Sub、多 AOF 文件 |
---
## 十、运维要点
- **内存：** 设置 maxmemory，监控内存使用率
- **连接数：** 合理设置 maxclients，使用连接池
- **持久化：** 生产环境建议 AOF + RDB 混合持久化
- **备份：** 定期备份 RDB 文件到远程存储
- **监控指标：** QPS、内存使用、连接数、命中率、慢查询
- **安全：** 设置密码、禁用危险命令（FLUSHALL、KEYS）、绑定内网 IP