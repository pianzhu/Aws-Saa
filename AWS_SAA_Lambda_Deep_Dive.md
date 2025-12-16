# AWS Certified Solutions Architect - Associate (SAA) - Lambda Deep Dive

## 1. 核心概念与架构原则 (Core Concepts & Architecture)

AWS Lambda 是 AWS **Serverless** 计算的核心服务。
*   **Event-driven (事件驱动)**：代码仅在触发时运行。
*   **按量付费**：按请求数量和执行时间（毫秒级）计费。
*   **无服务器管理**：无需管理 OS、补丁或扩展。

> [!IMPORTANT]
> **SAA 核心考点原则：**
> *   **最长运行时间**：15 分钟。超过此限制的认为不适合用 Lambda（改用 Fargate 或 Step Functions）。
> *   **临时性**：Lambda 是无状态的。持久化数据必须存放在 S3, DynamoDB, EFS 等外部存储。
> *   **磁盘空间**：`/tmp` 目录（最大 10GB）用于并在执行期间临时缓存。

---

## 2. 并发与扩缩容 (Concurrency & Scaling)

### 并发限制 (Concurrency Limits)
*   **Account Level (账户级别)**: 默认每个 Region **1000** 个并发执行。这是一个**软限制 (Soft Limit)**，可以申请提升。
*   **Function Level (函数级别) - Reserved Concurrency (预留并发)**:
    *   **保证资源**：为关键函数预留特定的并发配额，确保不受其他函数抢占（"嘈杂邻居"问题）。
    *   **限制上限**：也可以用来**限制**函数的最大并发数（例如：防止 Lambda 压垮下游的 RDS 数据库，将并发设为 50）。
    *   **Kill Switch**：将预留并发设为 **0** 可以完全停止函数运行。

### 冷启动 (Cold Start) vs 预置并发 (Provisioned Concurrency)
*   **Cold Start (冷启动)**: 当函数很久未运行或正在扩展新实例时，AWS 需要初始化环境（下载代码、启动容器、初始化 Runtime）。这会导致第一次请求延迟增加。
    *   *常见场景*：Java/C# 运行时较慢，VPC 连接（旧版）较慢。
*   **Provisioned Concurrency (预置并发)**:
    *   **解决冷启动的终极方案**。AWS 提前初始化并保持环境“热”状态。
    *   **适用场景**：对延迟极其敏感的生产环境。
    *   **成本**：比按需并发贵，因为你要为预热的资源付费。

> [!TIP]
> **Exam Tip**: 题目如果提到“消除 Lambda 函数的启动延迟”或“高性能要求”，首选 **Provisioned Concurrency**。

---

## 3. 配置与安全 (Configuration & Security)

### 环境变量 (Environment Variables)
*   用于配置函数行为（如 DB 连接串、API Key），无需修改代码。
*   **加密 (Encryption)**:
    *   默认使用 AWS 托管 Key 进行静态加密。
    *   可以使用 **KMS (CMK)** 进行客户管理的加密。
    *   在代码中通过 API 调用解密敏感数据。

### 权限管理 (IAM Permissions)
1.  **Execution Role (执行角色)**:
    *   **Who**: Lambda 函数本身。
    *   **What**: 允许 Lambda 访问其他 AWS 资源（例如：`s3:GetObject`, `dynamodb:PutItem`, `logs:CreateLogGroup`）。
    *   *注意*：必须有 CloudWatch Logs 权限才能写日志。
2.  **Resource-based Policy (资源策略)**:
    *   **Who**: 谁可以触发 Lambda？
    *   **What**: 允许其他服务（如 S3, SNS）或账户调用此 Lambda。
    *   *命令*：`aws lambda add-permission`。

### VPC 集成 (VPC Integration)
默认情况下，Lambda 运行在 AWS 管理的 VPC 中，可以访问互联网但不能访问你的私及有子网资源（如 RDS）。
*   **连接 VPC**: 需要配置 **Subnets** 和 **Security Groups**。
*   **ENI (Hyperplane ENI)**: AWS 现在使用 Hyperplane ENI，大大减少了 VPC Lambda 的冷启动时间（不再为每个容器创建 ENI）。
*   **互联网访问**: 连接到 VPC 的 Lambda **默认失去互联网访问权限**。
    *   **解决方案**：必须在公有子网配置 **NAT Gateway**，并将 Lambda 部署在私有子网，路由指向 NAT GW。
    *   *错误观念*：给 Lambda 分配公网 IP 是没用的，因为它没有 IGW 路由。

---

## 4. 管理与部署 (Management & Deployment)

### Lambda Layers (层)
*   **依赖管理**: 将依赖包（如 `numpy`, `pandas`, 自定义 SDK）打包成 Layer。
*   **优势**: 减小部署包体积，促进代码复用，分离业务逻辑与依赖库。

### 版本与别名 (Versions & Aliases)
1.  **Versions (版本)**:
    *   **Immutable (不可变)**: 一旦发布，版本代码和配置不能修改。
    *   `$LATEST`: 指向最新可变代码的特殊版本。
2.  **Aliases (别名)**:
    *   **Mutable (可变)**: 指向特定版本的指针（如 `PROD` -> `v1`, `DEV` -> `$LATEST`）。
    *   **Traffic Shifting (流量加权)**: 
        *   支持**蓝绿部署**或**金丝雀发布**。
        *   例如：将 `PROD` 别名配置为 90% 流量去 `v1`，10% 流量去 `v2`。

> [!WARNING]
> **Exam Tip**: CodeDeploy 可以自动化 Lambda 的流量转移（Canary, Linear, All-at-once），但底层机制是利用 **Aliases**。

---

## 5. 触发器与集成 (Invocations & Integration)

### 触发模式
1.  **Synchronous (同步)**: 客户端等待响应。
    *   *例子*: **API Gateway**, **ALB**, CLI invoke。
2.  **Asynchronous (异步)**: 客户端不等待，AWS 虽然排队处理。
    *   *例子*: **S3**, **SNS**, EventBridge。
    *   *重试*: 默认重试 2 次。
3.  **Event Source Mapping (轮询/流)**: Lambda 轮询流数据。
    *   *例子*: **SQS**, **Kinesis**, DynamoDB Streams。
    *   *批处理*: 将多条记录合并为一个 Event 批次处理，提高效率。

### 失败处理：DLQ vs Destinations
*   **Dead Letter Queue (DLQ)**:
    *   旧机制。只支持 SNS 或 SQS。
    *   仅用于**异步**调用失败后。
*   **Destinations (目标)**:
    *   新机制，更强大。
    *   支持 **Success** 和 **Failure** 两种结果。
    *   支持目标更多：SQS, SNS, Lambda, EventBridge。
    *   提供详细的执行上下文（错误堆栈等）。

> [!NOTE]
> 对于 **SQS** 触发 Lambda 失败的情况，消息会回退到 SQS 队列（由 Visibility Timeout 控制），最终由 SQS 的 DLQ 处理，而不是 Lambda 的 DLQ。

---

## 6. 限制与性能 (Limits & Performance)

*   **Memory (内存)**: 128 MB 到 10 GB。
    *   **CPU**: 无法单独设置 CPU。**CPU 算力与内存成正比**。如果计算密集型任务慢，**增加内存**也会增加 CPU。
*   **Timeout (超时)**: **15 分钟 (900 秒)** 硬限制。
    *   *对策*: 长时间运行任务应使用 AWS Step Functions 或 ECS/Batch。
*   **Payload Size**:
    *   同步调用：6 MB。
    *   异步调用：256 KB。

---

## 7. 边缘计算 (Edge Computing)

### Lambda@Edge vs CloudFront Functions

| 特性 | CloudFront Functions | Lambda@Edge |
| :--- | :--- | :--- |
| **运行时** | 仅 JavaScript (轻量级) | Node.js, Python |
| **执行位置** | 218+ 边缘节点 (最边缘，类似 CDN 缓存节点) | 13+ 区域边缘缓存 (Regional Edge Caches) |
| **启动延迟** | < 1 ms (无冷启动，极速) | 有冷启动 (毫秒到秒级) |
| **最大执行时间** | < 1 ms | 5 秒 (Viewer Request/Response)<br>30 秒 (Origin Request/Response) |
| **用例** | 简单的 Header 操作、URL 重写、Token 校验 | 复杂的逻辑、访问外部网络/数据库、处理大文件 body |
| **网络访问** | **无** (不能访问互联网或 AWS 服务) | **有** (可以访问 Internet 和 AWS 服务) |

> [!IMPORTANT]
> **Exam Tip**: 
> *   简单 HTTP header 修改/重定向 -> **CloudFront Functions** (更便宜、更快)。
> *   需要查数据库、第三方 API调用、处理复杂逻辑 -> **Lambda@Edge**。

---

## 8. SAA 考试常见场景总结 (Scenario Cheat Sheet)

1.  **大文件处理**: S3 上传触发 Lambda。如果文件处理超过 15 分钟 -> **S3 Event -> SQS -> EC2/ECS**，或者利用 **Step Functions** 编排。
2.  **解耦架构**: 总是优先考虑中间件。**API Gateway -> SQS -> Lambda** (高并发缓冲)。
3.  **私有资源访问**: Lambda 访问 RDS -> 必须在 VPC 内 -> 需要访问公网/DynamoDB -> 需要 NAT Gateway / VPC Endpoint。
4.  **成本优化**: 
    *   如果 Lambda 调用频率极高且逻辑极其简单 -> 考虑 API Gateway 直接集成服务 (无需 Lambda) 或 CloudFront Functions。
    *   Arm 架构 (Graviton2) 通常比 x86 性价比高 20%。
5.  **数据库连接耗尽**: Lambda 扩容太快耗尽 RDS 连接数 -> 必须使用 **RDS Proxy**。

