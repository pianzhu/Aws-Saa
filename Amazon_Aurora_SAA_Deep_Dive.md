# Amazon Aurora SAA Deep Dive

Amazon Aurora 是 AWS 专为云构建的关系型数据库引擎，完全兼容 MySQL 和 PostgreSQL。它在保留开源数据库简单性和成本效益的同时，提供了商业级数据库的速度和可用性（MySQL 5倍性能，PostgreSQL 3倍性能）。

## 1. 核心架构与存储 (Core Architecture & Storage)

Aurora 与传统 RDS 最本质的区别在于**计算与存储分离**。

*   **Cluster Volume (集群卷)**：
    *   Aurora 数据存储在虚拟的集群卷中，由 SSD 驱动。
    *   **自动扩展**：存储容量从 10GB 起步，随数据增加自动扩展，最大支持 **128 TB**。无需预置存储。
    *   **6路复制跨3个AZ**：数据被分成 10GB 的保护组（Protection Groups）。每个保护组有 6 个副本，分布在 3 个可用区（AZ）中（每个 AZ 2 个副本）。
    *   **Quorum 模型**：
        *   **写quorum**：需要 4/6 个副本确认。
        *   **读quorum**：需要 3/6 个副本确认（加上修复能力）。
    *   **容错能力**：
        *   可以承受 **失去 1 个整个 AZ** 而不丢失数据且不影响写入。
        *   可以承受 **失去 2 个整个 AZ** 而不丢失数据（依然可读）。
    *   **自愈 (Self-Healing)**：后台持续扫描数据块错误并自动从其他副本修复。

## 2. 实例与端点 (Instances & Endpoints)

Aurora 集群包含不同角色的实例：

*   **Primary Instance (主实例)**：支持读写操作（R/W）。每个集群通常只有 1 个（Multi-Master 除外）。
*   **Aurora Replica (只读副本)**：只支持读取操作（Read-Only）。
    *   最多支持 **15** 个副本（RDS 只能 5 个）。
    *   **共享存储**：副本与主实例共享同一个底层存储卷，因此**几乎没有复制延迟**（通常 < 10-20ms），且创建副本极快。
    *   **故障转移目标**：如果主实例挂了，Aurora 会自动将一个副本提升为新的主实例。

**连接端点 (Endpoints)**：
1.  **Cluster Endpoint (集群端点)**：指向当前的主实例 (Primary)。应用程序用于**写入**。常用于写操作或不知道谁是主库时。
2.  **Reader Endpoint (读者端点)**：自动对所有 Aurora Replicas 进行 DNS 轮询负载均衡 (Load Balancing)。应用程序用于**读取**。
3.  **Custom Endpoint (自定义端点)**：用户自定义的端点，指向特定的副本子集（例如：将高性能实例分为一组用于分析，低性能实例分为一组用于内部报表）。避免分析查询影响生产流量。
4.  **Instance Endpoint (实例端点)**：直接指向特定实例。不建议在应用中硬编码。

## 3. 高可用与故障转移 (HA & Failover)

*   **自动故障转移**：
    *   如果主实例不可用，Aurora 会在 **30秒内** 自动故障转移（如果有副本）。
    *   **Tier (优先级)**：你可以为每个副本定义优先级 (Tier 0 - Tier 15)。Tier 0 优先级最高，故障转移时首选。
    *   如果**没有**副本，Aurora 将尝试在同一 AZ 创建新实例（可能会有停机时间）。
*   **缓存生存 (Cluster Cache Management)**：在故障转移时，Aurora 可以尝试将之前的缓存热度保留，确保新主库确立后性能不剧烈下降（仅限特定版本 PostgreSQL）。

## 4. 关键高级特性 (Advanced Features)

### 4.1. Aurora Global Database (全局数据库)
*   **跨区域灾备 (DR)**：一个主区域进行读写，最多 5 个辅助区域（Secondary Regions）只读。
*   **低延迟**：通过专用的物理复制与 AWS 骨干网，在此场景下跨区域复制延迟通常 **< 1 秒**。
*   **RTO**：在区域级故障发生时，辅助区域可以在 **< 1 分钟** 内提升为新的读写主区域（Review options carefully: RPO 接近 0，RTO < 1 min）。
*   **用途**：低延迟全球读取、灾难恢复。

### 4.2. Aurora Serverless (v1 & v2)
*   **按需扩展**：根据负载自动启动、扩展和关闭数据库容量（ACU - Aurora Capacity Units）。
*   **Serverless v1**：适合不频繁、不可预测的工作负载。如果不使用，可以缩减到 0（暂停）。
*   **Serverless v2**：专为所有类型的工作负载设计（包括生产环境）。**毫秒级** 扩缩容（Scale up/down instantly）。支持细粒度的 ACU 调整（0.5 ACU 起步）。
*   **考点**：如果是“不可预测的流量”或“开发/测试环境希望省钱（不跑时不收费）”，选 Serverless。

### 4.3. Aurora Multi-Master (多主集群)
*   **多点写入**：集群中所有节点都可以进行读写操作。
*   **持续可用性**：如果有节点故障，应用可以立即使用其他节点写入，实现零停机时间。
*   **限制**：目前只支持 MySQL，且功能有限制。
*   **考点**：极高的写入可用性要求（即使有微小停机也不能接受）。

### 4.4. Cloning (克隆)
*   **Copy-on-Write (写时复制)**：创建新集群基于现有集群的数据，**速度极快**（几分钟），且初始**不占额外空间**（只有数据变更后才占空间）。
*   **场景**：使用生产数据快速创建测试环境 (Staging/Test)。

### 4.5. Backtrack (回溯)
*   **时光倒流**：允许将数据库状态“倒带”到过去某个时间点，而无需从备份恢复。
*   **原理**：不依赖快照，直接在日志层面操作。速度极快。
*   **限制**：仅限 MySQL 兼容版。需要预先开启（会增加成本）。
*   **场景**：开发人员误删了数据（DROP TABLE），需要立即撤销操作。

### 4.6. Parallel Query (并行查询)
*   **计算下推**：将查询处理逻辑下推到存储层节点，利用成千上万个存储节点的 CPU 并行处理。
*   **场景**：大幅提升分析型查询（OLAP 风格的查询）在 Aurora 上的性能。

### 4.7. Machine Learning Integration
*   通过 SQL (`FROM mysql_ml` 或 PostgreSQL 扩展) 直接调用 SageMaker 或 Comprehend 模型。
*   场景：在数据库即时进行欺诈检测、情感分析。

## 5. 架构最佳实践 (Architectural Pillars)

### 5.1. 成本优化 (Cost Optimization)
*   **Serverless**：对于间歇性工作负载，使用 Serverless v2 自动缩放，避免为空闲资源付费。
*   **Stop/Start**：如果不使用 Serverless，开发环境可以在非工作时间暂时停止集群（最多7天）。
*   **I/O 优化**：Aurora 收费除了实例和存储，还有 **I/O 请求**。
    *   *Aurora I/O-Optimized* 配置：对于 I/O 密集型应用，选择此配置可节省高达 40% 成本（虽然存储单价稍高，但免除 I/O 费用）。

### 5.2. 高可用与容错 (High Availability & Fault Tolerance)
*   **Multi-AZ**：虽然存储自动 Multi-AZ，但计算层要 HA 必须至少有 1 个 Replica 在不同 AZ。
*   **自动修复**：利用 Aurora 存储的自我修复特性，无需人为干预坏块。
*   **Global Database**：针对区域级故障的终极 HA 方案。

### 5.3. 性能 (Performance)
*   **读写分离**：应用程序应配置写请求走 Cluster Endpoint，读请求走 Reader Endpoint。
*   **Custom Endpoint**：针对不同业务类型隔离流量，防止报表查询拖慢在线交易。
*   **连接池 (Proxy)**：使用 **RDS Proxy** 来管理数据库连接，特别是配合 Lambda 使用时，减少连接建立开销和处理突发流量。

### 5.4. 安全 (Security)
*   **加密**：支持 KMS 静态加密（创建时开启），无法对未加密集群直接加密（需快照-复制-恢复）。
*   **IAM Auth**：使用 IAM 角色和 Token 进行数据库认证，无需在代码中硬编码密码。
*   **安全组**：控制网络访问。私有子网部署的最佳实践。

## 6. SAA 常见考点对比分析 (Comparative Analysis)

| 特性 | Aurora | RDS (MySQL/PostgreSQL) | 说明 |
| :--- | :--- | :--- | :--- |
| **存储扩展** | 自动扩展，最大 128TB | 手动/自动扩展，最大 64TB | Aurora 管理更省心，上限更高。 |
| **副本数量** | 最多 15 个 Aurora Replicas | 最多 5 个 Read Replicas | Aurora 读扩展能力更强。 |
| **副本类型** | 物理复制（共享存储），延迟极低 | 逻辑复制（Binlog），有一定延迟 | Aurora 副本对性能影响几乎为 0。 |
| **Multi-AZ** | 存储层原生 Multi-AZ，计算层靠 Replicas | 也是 Multi-AZ，但 Standby 不可读 | **关键点**：RDS Standby 是闲置的；Aurora Replica 是活跃可读的。 |
| **故障转移** | < 30 秒 (CNAME 切换) | 60-120 秒 (DNS 变更) | Aurora 恢复更快。 |
| **跨区域** | Global Database (<1s 延迟) | Cross-Region Read Replica | Global Database 性能和 RTO 远优于普通跨区副本。 |

## 7. 考试关键字 (Exam Keywords)

*   **"High performance", "MySQL compatible", "Auto-scaling storage"** -> **Amazon Aurora**。
*   **"Serverless", "Infrequent usage", "Variable workloads"** -> **Aurora Serverless**。
*   **"Disaster Recovery with low RTO/RPO across regions"** -> **Aurora Global Database**。
*   **"Roll back database to a point in time quickly without restore"** -> **Backtrack** (仅 MySQL)。
*   **"Testing with production data"** -> **Cloning**。
*   **"Separate analytical queries from transactional traffic"** -> **Custom Endpoints**。
*   **"Optimize costs for predictable high I/O app"** -> **Aurora I/O-Optimized**。
