# Amazon DynamoDB SAA Deep Dive

面向 AWS Solutions Architect Associate (SAA) 考试的 Amazon DynamoDB 深度解析。本指南围绕**成本优化**、**高可用性**、**安全性**和**性能**四大支柱，结合考试常见场景进行详细拆解。

---

## 1. 核心架构与基本概念 (Core Architecture)

### 数据模型 (Data Model)
- **Table (表)**: 数据的集合。
- **Item (项目)**: 类似于关系型数据库中的行 (Row)，但无模式 (Schema-less)。
- **Attribute (属性)**: 类似于列 (Column)，可以是标量或嵌套结构。

### 主键设计 (Primary Keys) - **考试重点**
DynamoDB 不支持复杂的 Join 操作，通过主键设计来满足查询模式。

1.  **分区键 (Partition Key / HASH)**:
    - 仅由一个属性组成。
    - DynamoDB 使用此键的哈希值决定数据存储在哪个物理分区。
    - **查询限制**: 只能通过精确匹配 (`Equality`) 查询。
    - **最佳实践**: 选择基数高（Cardinality）的属性以避免热分区（如 `UserID`, `SessionID`）。

2.  **复合主键 (Composite Key: Partition Key + Sort Key / HASH + RANGE)**:
    - 由分区键 + 排序键组成。
    - 相同分区键的数据存储在一起，并按排序键排序。
    - **查询能力**: 分区键必须精确匹配，排序键支持范围查询 (`>`,`<`, `between`, `begins_with`)。
    - **场景**: 这里的 `DeviceID` 是分区键，`Timestamp` 是排序键。

### 分区与热分区 (Partitions & Hot Partitions)
- **原理**: DynamoDB 根据分区键的哈希值将数据分散到多个物理分区。
- **热分区问题 (Hot Partition Issue)**:
    - 如果大量读/写请求集中在少数几个分区键上，会导致该分区的吞吐量耗尽（`ProvisionedThroughputExceededException`），即使表总容量未满。
- **解决方案**:
    - **Write Sharding (写入分片)**: 在分区键后添加随机数或计算后缀（如 `UserID#1`, `UserID#2`），将流量打散。
    - **高基数键**: 避免使用只有少量取值的字段（如 `Gender`, `Status`）作为分区键。

---

## 2. 索引设计 (Indexes: LSI vs GSI) - **高频考点**

考试中经常要求辨析选择哪种索引来满足新的查询需求。

| 特性 | 本地二级索引 (LSI - Local Secondary Index) | 全局二级索引 (GSI - Global Secondary Index) |
| :--- | :--- | :--- |
| **创建时机** | **仅能在创建表时创建** (不可后期添加/修改) | **随时**可以创建或删除 |
| **分区键** | **必须与基表相同** | **可以与基表不同** (完全新的分区键) |
| **排序键** | 必须与基表不同 | 可以存在，也可以不存在 |
| **容量消耗** | 共享基表的读写容量 (RCU/WCU) | 有**独立**的读写容量设置 (Provisioned/On-demand) |
| **一致性** | 支持强一致性 (Strongly Consistent) 和最终一致性 | **仅支持最终一致性 (Eventual Consistency)** |
| **属性投影** | KEYS_ONLY, INCLUDE, ALL | KEYS_ONLY, INCLUDE, ALL |
| **限制** | 每个分区键下的 Item 集合大小限制为 10GB | 无大小限制 |

**SAA 答题技巧**:
- 如果题目说“需要对已存在的表添加索引”，直接选 **GSI**。
- 如果题目要求“强一致性读取索引”，只能选 **LSI**。
- 如果查询模式完全改变（需要用不同的字段作为分区键），必须选 **GSI**。

---

## 3. 容量模式与成本优化 (Capacity Modes)

### 按需容量模式 (On-Demand Mode)
- **场景**: 流量不可预测、突发性强、新应用（未知流量模型）。
- **计费**: 按读写请求次数付费 (Pay-per-request)。
- **特点**: 无需容量规划，自动适应负载。
- **成本**: 单次请求单价较高，但无闲置成本。

### 预配置容量模式 (Provisioned Mode)
- **场景**: 流量可预测、平稳、逐渐增长的业务。
- **计费**: 按设置的 RCU (Read Capacity Units) 和 WCU (Write Capacity Units) 小时数付费。
- **自动扩展 (Auto Scaling)**: 必须开启。当利用率达到阈值（如 70%）时自动增加容量。
- **成本**: 充分利用时，单位成本比 On-Demand 低。
- **保留容量 (Reserved Capacity)**: 类似于 RI，承诺 1 年或 3 年可获得大幅折扣（甚至 > 50%）。

---

## 4. 读写一致性 (Consistency Models)

- **最终一致性 (Eventual Consistency)**:
    - 默认模式。读取可能不会立即反映刚刚完成的写入（通常 1 秒内一致）。
    - 消耗 **0.5 RCU** (4KB/item)。最省钱。
- **强一致性 (Strongly Consistent)**:
    - 保证读取到最新的写入。
    - 消耗 **1.0 RCU** (4KB/item)。成本翻倍。
    - 只有 `GetItem`, `Query`, `Scan` 支持，GSI 不支持。

---

## 5. 高级功能与性能优化 (Advanced Features)

### DynamoDB Accelerator (DAX)
- **定义**: DynamoDB 专用的全托管、高可用**内存缓存**集群。
- **性能**: 将响应时间从毫秒 (ms) 级降低到**微秒 (μs)** 级。
- **场景**: 读多写少、对延迟极端敏感的场景（如实时竞价、游戏排行榜）。
- **特点**: API 兼容（应用层代码改动极小），Write-through（直写）策略。
- **对比**: ElastiCache 需要应用层维护缓存逻辑（Lazy Loading/Write Aside），DAX 是透明的。

### DynamoDB Streams (流)
- **定义**: 按时间顺序捕获表中 Item 的修改事件（INSERT, MODIFY, REMOVE）。
- **保留期**: 数据保留 **24 小时**。
- **应用场景**:
    - 触发 Lambda 函数执行业务逻辑（如注册后发送邮件）。
    - 实时数据分析/复制到 Kinesis。
    - 跨区域复制的基础（Global Tables 的底层机制）。
- **视图类型**:
    - `KEYS_ONLY`: 仅键。
    - `NEW_IMAGE`: 修改后的完整项。
    - `OLD_IMAGE`: 修改前的完整项。
    - `NEW_AND_OLD_IMAGES`: 修改前后的完整项。

### Global Tables (全局表)
- **架构**: 基于 DynamoDB Streams 构建的**多区域 (Multi-Region)**、**主动-主动 (Active-Active)** 复制架构。
- **场景**:
    - 灾难恢复 (DR) - 区域级容灾。
    - **本地读写性能** - 全球用户就近读写。
- **冲突解决**: Last Writer Wins (最后写入者胜)。
- **要求**: 必须开启 Streams，必须为空表（创建时）。

---

## 6. 备份与恢复 (Backup & Recovery)

### Point-in-Time Recovery (PITR - 时间点恢复)
- **机制**: 连续备份，保护过去 **35 天**的数据。
- **恢复**: 可以恢复到 35 天窗口内的**任何一秒**。
- **场景**: 防止意外删除或写入操作（人为错误/脚本错误）。
- **注意**: 恢复不仅是在原表，而是创建一个**新表**。

### On-Demand Backup (按需备份)
- **机制**: 手动触发的完整备份。
- **保留期**: 长期保留（可永久）。
- **AWS Backup 集成**: 支持跨区域复制备份、跨账户复制备份（合规性要求）。
- **性能**: 备份过程不影响表的性能。

---

## 7. 安全性 (Security)

- **静态加密 (Encryption at Rest)**:
    - 默认开启 KMS 加密。
    - 选项: AWS Owned Key (默认/免费), AWS Managed Key, Customer Managed Key (CMK)。
- **传输加密 (Encryption in Transit)**: 仅支持 HTTPS 端点。
- **访问控制**:
    - IAM Policy: 控制谁能访问 DynamoDB API。
    - **Fine-Grained Access Control (细粒度访问控制)**: 利用 `dynamodb:LeadingKeys` 条件键，限制用户只能访问主键匹配其 UserID 的行（例如：移动应用用户只能读取自己的数据）。
- **网络安全**: 使用 **VPC Gateway Endpoint** (注意不是 Interface Endpoint) 在 VPC 内部私密访问 DynamoDB。

---

## 8. 开发与操作特性 (Dev & Ops)

### TTL (Time To Live)
- **功能**: 定义一个属性作为过期时间戳（Epoch Time），DynamoDB 会自动在后台删除过期的 Item。
- **成本**: **免费**删除操作（且不消耗 WCU）。
- **场景**: Session 存储、临时日志、购物车清理。
- **注意**: 删除不是实时的，通常在过期后 48 小时内完成。

### 事务 (Transactions)
- **API**: `TransactWriteItems`, `TransactGetItems`。
- **特性**: 提供 **ACID** 保证。支持跨多个表、跨多个 Item 的原子性操作。
- **成本**: 消耗 2 倍的 RCU/WCU（准备 + 提交）。
- **场景**: 金融转账（A 扣钱，B 加钱，必须同时成功或失败）。

### 批量操作 (Batch Operations)
- **API**: `BatchGetItem`, `BatchWriteItem`。
- **优势**: 减少网络往返 (Round Trips)，提高吞吐量。
- **注意**: 不保证事务性，部分成功部分失败需要重试（Exponential Backoff）。

### 条件写入 (Conditional Writes) / 乐观锁
- **功能**: 写入时检查条件，如果条件不满足则失败。
- **场景**:
    - **防止覆盖 (Prevent Overwrite)**: `attribute_not_exists(PK)`。
    - **乐观锁 (Optimistic Locking)**: 更新前检查版本号 (`version = expected_version`)。

---

## 9. 常见场景对比总结 (Comparisons)

### DynamoDB vs RDS vs Aurora
| 维度 | DynamoDB | RDS / Aurora |
| :--- | :--- | :--- |
| **数据模型** | NoSQL (键值/文档) | 关系型 (SQL) |
| **Schema** | Schema-less (灵活) | 固定 Schema |
| **Join/复杂查询**| 不支持 (Limited) | 原生支持 |
| **扩展性** | 水平扩展 (无限存储/吞吐) | 垂直扩展 (Aurora 可水平读) |
| **使用场景** | 高并发、低延迟、简单查询、海量数据 | 复杂关系、事务密集、传统 ERP/CRM |

### Scan vs Query
- **Query (查询)**:
    - 基于 Partition Key (+ Sort Key) 查找。
    - **高效**。直接定位数据。
- **Scan (扫描)**:
    - 扫描**整张表**。
    - **低效且昂贵**。消耗大量 RCU。
    - **优化**: 尽量避免使用。如果必须用，考虑使用 **Parallel Scan** (并行扫描) 提高速度（但消耗更多 RCU）。

---
**记住核心原则**: 看到“Key-Value”、“高并发”、“毫秒级延迟”、“无模式”、“海量数据扩展” -> **首选 DynamoDB**。
