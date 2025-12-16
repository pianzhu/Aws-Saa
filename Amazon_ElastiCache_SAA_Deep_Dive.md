# Amazon ElastiCache SAA Deep Dive

> [!IMPORTANT]
> **考试核心原则 (Exam Core Principles)**
> - **性能 (Performance)**: 内存级别亚毫秒级延迟，减轻数据库负载。
> - **高可用 (HA)**: Redis 多可用区 (Multi-AZ) 自动故障转移。
> - **成本 (Cost)**: 正确选择节点类型，使用分层存储 (Data Tiering) 优化成本。
> - **架构 (Architecture)**: 读写分离，缓存策略选择。

---

## 1. Redis vs Memcached 核心对比

这是 SAA 考试中最常见的基础考点，必须熟练掌握两者的适用场景。

| 特性 | **Redis (Remote Dictionary Server)** | **Memcached** | **SAA 考点解析** |
| :--- | :--- | :--- | :--- |
| **数据结构** | **丰富**：Strings, Hashes, Lists, Sets, Sorted Sets, Bitmaps, Geospatial, HyperLogLog | **简单**：纯 Key-Value 存储 (Strings, Objects) | 如果题目提到“排行榜(Sorted Sets)”、“地理位置(Geo)”、“计数器”，必选 **Redis**。简单对象缓存可选 Memcached。 |
| **持久性** | **支持**：RDB 快照 (Snapshots) 和 AOF (Append Only File) | **不支持**：纯内存，重启数据丢失 | 如果需要“数据持久化”、“备份与恢复”，必选 **Redis**。 |
| **高可用与复制** | **支持**：主从复制 (Replication)，多 AZ 自动故障转移 (Multi-AZ Auto-Failover) | **不支持**：无复制功能，节点故障意味着这部分缓存丢失 | 如果需要“高可用”、“读写分离”、“自动容错”，必选 **Redis**。 |
| **扩展性** | **垂直 & 水平**：支持集群模式 (Cluster Mode) 进行分片 (Sharding) | **水平**：多线程架构，通过增加节点水平扩展 | Memcached 只有多线程这一个优势，但在 SAA 考题中极少作为决定性因素。 |
| **Pub/Sub** | **支持**：发布/订阅模式 | **不支持** | 消息传递场景选 Redis。 |
| **适用场景** | 复杂缓存、排行榜、会话存储 (Session Store)、地理空间应用、需要持久化的场景 | 简单的页面缓存、数据库查询结果缓存、无需持久化且通过多线程处理大吞吐量的极简场景 | **即使是简单的 Session Store，AWS 最佳实践通常也推荐 Redis，因为其支持高可用。** |

> [!TIP]
> **考试技巧**: 看到 "Multi-threaded" (多线程) -> 选 **Memcached**。看到 "Sorted Sets", "Ranking", "Leaderboard", "Persistence", "Replication", "High Availability" -> 选 **Redis**。

---

## 2. ElastiCache for Redis 架构模式

### 2.1 非集群模式 (Cluster Mode Disabled)
- **结构**: 1 个分片 (Shard)，包含 1 个主节点 (Primary) 和最多 5 个只读副本 (Read Replicas)。
- **操作**: 所有写操作都在 Primary，读操作可分发到 Replicas。
- **故障转移**: 如果启用 Multi-AZ，Primary 挂掉会自动提升一个 Replica 为新 Primary。
- **适用**: 数据量较小，主要是读密集型应用。

### 2.2 集群模式 (Cluster Mode Enabled) - 分片 (Sharding)
- **结构**: 数据分布在多个分片 (Shards) 上。每个分片有自己的 Primary 和 Replicas。
- **扩展性**: 支持水平扩展 (Scale Out)，最大支持 500 个分片 (取决于配置)。
- **性能**: 写入负载分布在多个分片上，突破单节点写入瓶颈。
- **适用**: **写密集型 (Write-intensive)** 应用，海量数据存储（超过单个节点内存限制）。

### 2.3 读副本与高可用 (Read Replicas & HA)
- **异步复制**: Primary 到 Replica 的复制是异步的 (Asynchronous)。
- **故障转移 (Failover)**:
    - 必须启用 **Multi-AZ**。
    - 检测到 Primary 故障 -> 自动提升 Replica -> 更新 DNS 入口 (无需修改应用代码)。
- **手动故障转移**: 可以用于测试。

---

## 3. 安全最佳实践 (Security)

### 3.1 认证与授权 (Authentication & Authorization)
- **Redis AUTH**: 传统的 Redis 密码验证。
- **RBAC (Role-Based Access Control)**: Redis 6.0+ 支持。创建用户和用户组，精细控制权限（如允许只读，禁止 `FLUSHALL` 命令）。
- **IAM Auth**: ElastiCache for Redis 7.0+ 支持使用 IAM 身份验证（无需管理密码）。
- **Memcached**: 仅支持 SASL 认证。

### 3.2 加密 (Encryption)
- **At Rest (静态加密)**: 使用 KMS CMK 加密磁盘上的数据（和快照）。
- **In Transit (传输中加密)**: 支持 TLS/SSL。
    - 启用 TLS 后，客户端连接必须支持 SSL。
    - 使用 `redis-cli` 连接时需加 `--tls` 参数。

### 3.3 网络安全
- **VPC 内部**: ElastiCache 运行在 VPC 私有子网中，不公开公网 IP。
- **Security Groups**: 通过安全组控制访问（例如：只允许 App Server 的安全组访问 6379 端口）。

---

## 4. 备份与恢复 (Backup & Restore)

- **快照 (Snapshots)**:
    - 仅 **Redis** 支持 (Memcached 不支持)。
    - 可以手动创建，也可以自动备份 (设置备份窗口)。
    - 备份存储在 S3 中（由 AWS 管理，不可直接访问，但可导出）。
    - **性能影响**: 建议在 Read Replica 上进行备份，通过 `reserved-memory` 参数预留内存，防止备份时发生 Swap 导致性能下降。
- **恢复**: 从快照恢复会创建一个**新的** ElastiCache 集群。

---

## 5. Redis Global Datastore (全球数据存储)

- **场景**: 跨 Region 容灾 (DR) 和 本地低延迟读取。
- **架构**:
    - 1 个主 Region (读写)。
    - 最多 5 个从 Region (只读)。
- **复制延迟**: 跨 Region 复制通常在 1 秒以内。
- **灾难恢复**: 如果主 Region 挂了，可以将从 Region 提升为主 (DR 切换)。
- **考点**: "跨区域低延迟访问" 或 "Region 级高可用"。

---

## 6. 缓存策略 (Caching Strategies)

这是架构设计的核心，SAA 经常考察针对特定场景选择哪种策略。

### 6.1 延迟加载 (Lazy Loading / Cache-Aside)
- **流程**: 应用先查缓存 -> 没命中 (Miss) -> 查数据库 -> 写入缓存 -> 返回数据。
- **优点**: 仅缓存被请求的数据（节省内存）；节点故障不致命（数据在 DB 有）。
- **缺点**: 缓存未命中时有延迟（需查 DB）；数据可能过时 (Stale Data)。
- **适用**: 数据读取频率不高，且对实时性要求不严格的场景。

### 6.2 直写 (Write-Through)
- **流程**: 应用更新数据库的同时，立即更新缓存。
- **优点**: 缓存数据永远是最新的；读取速度快（几乎不 Miss）。
- **缺点**: 写入延迟增加（需写两处）；由于写多读少的数据也被缓存，可能浪费内存 (Cache Churn)。
- **适用**: 数据必须实时一致，且对写入延迟不敏感的场景。

### 6.3 组合策略与 TTL (Time To Live)
- **最佳实践**: 通常结合使用 Lazy Loading 和 Write-Through。
- **TTL (过期时间)**: 必须为 Key 设置 TTL。
    - 解决 Lazy Loading 的数据过时问题。
    - 解决 Write-Through 的内存浪费问题（自动清除冷数据）。
- **通过 TTL 强制数据更新**。

---

## 7. ElastiCache Serverless

- **定义**: 2023 年推出的新选项，无需管理节点、分片或实例类型。
- **扩展性**: 自动垂直扩展 (内存/CPU) 和水平扩展 (读写吞吐)。
- **计费**: 按存储的数据量 (GB-hours) 和请求单元 (ECPU) 付费。
- **适用**: 流量波动巨大、不可预测的工作负载，或者不想进行容量规划的场景。
- **考点**: "最少运维开销 (Least Operational Overhead)"，"不可预测的流量模式"。

---

## 8. SAA 典型场景与反模式

| 场景 | 推荐方案 | 为什么？ |
| :--- | :--- | :--- |
| **减轻 RDS 读负载** | **ElastiCache (Redis/Memcached)** | 缓存热点数据，减少 DB IOPS 消耗。 |
| **存储用户 Session** | **ElastiCache for Redis** | 高可用，带持久化，TTL 自动过期 Session。 |
| **排行榜 (Gaming Leaderboard)** | **Redis (Sorted Sets)** | Redis 原生支持 `zrank`, `zadd` 等排序操作，效率极高。 |
| **复杂关系查询/Join** | **RDS / Aurora** | **不要**用 ElastiCache 做复杂关系查询，它只是 Key-Value。 |
| **海量冷数据归档** | **S3 / Glacier** | **不要**用 ElastiCache 存冷数据，内存太贵。 |
| **简单的多线程 Key-Value** | **Memcached** | 唯一可能选 Memcached 的理由是极其简单的模型且需要多线程。 |
| **需要 SQL 查询能力** | **RDS / Aurora** | ElastiCache 不支持 SQL。 |

> [!WARNING]
> **Key Eviction (驱逐策略)**: 当内存满了怎么办？
> - Redis 默认策略通常是 `volatile-lru` (删除设置了 TTL 且最近最少使用的 Key)。
> - 考试如果不问细节，只需记住：**内存满了会根据策略删除旧数据，或者报错 (OOM)**。可以通过 **Data Tiering (数据分层)** 将冷数据移到 SSD 来扩容（仅限特定实例系列）。
