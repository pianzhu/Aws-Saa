# AWS SAA 核心考点深度解析：Amazon ECS & EKS

本文档专为 AWS Solutions Architect Associate (SAA) 备考设计。我们将从**架构师视角**出发，紧扣**成本优化、高可用性、安全性、性能效率**四大支柱，对 ECS (Elastic Container Service) 和 EKS (Elastic Kubernetes Service) 的核心考点进行逐一拆解。

---

## 1. 计算层选择：EC2 启动类型 vs. AWS Fargate

这是 SAA 考试中关于容器最频繁出现的考点，核心在于**管理开销**与**成本**的权衡。

| 特性 | EC2 启动类型 (EC2 Launch Type) | AWS Fargate (Serverless) |
| :--- | :--- | :--- |
| **管理模型** | **自管基础设施**。你需要配置、修补、扩展 EC2 实例。 | **无服务器**。AWS 管理底层基础设施，你只需关注容器本身。 |
| **定价模式** | 按 EC2 实例运行时间付费（无论容器是否跑满）。 | 按任务（Task/Pod）请求的 vCPU 和内存资源量 + 运行时间付费。 |
| **适用场景** | 1. 极度成本敏感且通过预留实例(RI)/Spot 优化极好的场景。<br>2. 需要挂载特定存储（如 EBS 卷，虽然 Fargate 现也支持部分，但 EC2 更灵活）。<br>3. GPU 支持（Fargate 现已支持，但 EC2 选项更多）。 | 1. **降低运维负担**（首选）。<br>2. 流量波动大，需快速扩缩容。<br>3. 批处理任务。 |
| **SAA 考点直击** | 题目如果强调“**最小化运维开销 (Administrative Overhead)**”，选 **Fargate**。<br>题目如果强调“**利用现有 RI**”或“**深度控制底层 OS**”，选 **EC2**。 |

---

## 2. 核心资源对象：Task Definition vs. Container Definition

理解这层级关系对于理解配置至关重要。

*   **Task Definition (任务定义)**：
    *   它是**蓝图 (Blueprint)**。
    *   定义了应用由哪些容器组成（支持多容器模式，类似 K8s 的 Pod）。
    *   **关键配置**：总 CPU/内存限制、网络模式（Network Mode）、IAM 角色（Task/Execution Role）、数据卷挂载。
*   **Container Definition (容器定义)**：
    *   在任务定义内部。
    *   指定具体镜像（Image）、端口映射、环境变量、健康检查命令。

> **考试陷阱**：如果要更新应用镜像版本，必须创建**新版本的 Task Definition**，然后更新 Service 使用新版本。

---

## 3. 弹性伸缩：Service Auto Scaling & Cluster Auto Scaling

SAA 考试非常关注如何应对流量变化。

### Service Auto Scaling (任务/应用层)
*   **对象**：ECS Service 中的 Task 数量。
*   **机制**：基于 CloudWatch 指标（如 CPU 使用率、内存使用率、ALB 请求数）自动增加或减少 **Task 的数量**。
*   **Target Tracking Scaling Policy**（目标跟踪）：最推荐，例如“保持平均 CPU 利用率在 70%”。

### Cluster Capacity Provider (基础设施层)
*   **问题**：当 Service 扩容 Task 时，如果底层 EC2 集群资源不足怎么办？
*   **解决方案**：**ECS Capacity Providers (容量提供商)**。
*   **功能**：自动管理底层 EC2 Auto Scaling Group (ASG)。当 Task 处于 PENDING 状态（因资源不足）时，它会自动触发 ASG 启动新实例。它也支持 **Managed Scaling** 来缩容空闲实例以节省成本。

---

## 4. 网络与通信

### 容器间通信 & 服务发现 (Service Discovery)
*   **AWS Cloud Map**：ECS 与 Cloud Map 紧密集成。
    *   当你启用服务发现时，AWS 会自动在 Route 53 私有托管区域中注册服务名称。
    *   **场景**：Service A 想调用 Service B，只需访问 `http://service-b.namespace`，无需关心 IP 变化。
    *   **对比**：相比传统 ELB 做内部通信，Cloud Map 更轻量且无额外 ELB 成本，但 ELB 提供更好的流量分发和健康检查。

### 负载均衡集成 (ALB/NLB)
*   **Application Load Balancer (ALB)**：最常用。支持动态端口映射（Dynamic Port Mapping）。
*   **Network Load Balancer (NLB)**：用于超高性能或需要静态 IP 的场景。
*   **Fargate 模式**：
    *   **必须使用 awsvpc 网络模式**。
    *   每个 Task 都有自己的 ENI 和私有 IP。
    *   负载均衡器直接将流量转发到 **Task 的 IP**（Target Type = `ip`）。

---

## 5. 安全性：IAM 角色的职责分离

这是安全最佳实践（最小权限原则）的必考题。

| 角色类型 | Task Execution Role (任务执行角色) | Task Role (任务角色) |
| :--- | :--- | :--- |
| **执行者** | **ECS Agent (代理)** | **你的应用程序代码** |
| **用途** | 应用启动**前**的基础设施操作。 | 应用运行**时**的权限。 |
| **典型权限** | 1. 从 ECR **拉取镜像** (Pull Images)。<br>2. 将日志**推送到 CloudWatch Logs**。<br>3. 从 Secrets Manager/SSM Parameter Store 读取敏感数据注入环境变量。 | 1. 访问 S3 桶上传文件。<br>2. 操作 DynamoDB 表。<br>3. 发送 SQS 消息。 |
| **考点口诀** | “拉镜像、打日志”找 Execution Role；“代码调 AWS API” 找 Task Role。 |

---

## 6. 混合云与扩展：ECS Anywhere & EKS Anywhere

*   **场景**：客户有合规性要求，数据必须留在本地数据中心，但希望使用统一的云上控制平面。
*   **ECS Anywhere / EKS Anywhere**：允许你在本地服务器（On-Premises）上运行 ECS Task 或 EKS 集群，但通过 AWS 控制台统一管理。
*   **连接**：通过 AWS Systems Manager (SSM) Agent 建立信任和连接。

---

## 7. EKS 专项：Managed Node Groups (托管节点组)

在 EKS 中，控制平面（Control Plane）主要由 AWS 管理，但数据平面（Data Plane / Worker Nodes）传统上需要用户管理 EC2。

*   **Managed Node Groups**：
    *   AWS 自动预置和管理 EC2 节点（包括打补丁、更新）。
    *   极大地简化了 EKS 的运维工作（Operational Excellence）。
    *   支持 **Spot Instances**，只需简单的配置即可在节点组中混合使用 On-Demand 和 Spot 实例。

---

## 8. 成本优化大杀器：Fargate Spot

*   **概念**：利用 AWS 空闲算力运行 Fargate 任务，价格最高可享 **70% 折扣**。
*   **风险**：AWS 可能会在 **2 分钟** 通知后回收资源。
*   **最佳实践（架构设计）**：
    1.  **容错性 (Fault Tolerance)**：仅用于**无状态 (Stateless)**、可中断的任务。
    2.  **混合策略**：在 Capacity Provider Strategy 中配置 `Base`（基准）使用 Fargate On-Demand 保证最低可用性，`Weight`（权重）部分使用 Fargate Spot 来处理爆发流量。
    3.  **应用处理**：应用程序需要监听 `SIGTERM` 信号，在 2 分钟内完成优雅退出。

---

## 9. 考点总结与易混淆概念对比

| 概念 A | 概念 B | 关键区别 |
| :--- | :--- | :--- |
| **ECR (Elastic Container Registry)** | **ECS (Elastic Container Service)** | ECR 是存镜像的仓库（类似 Docker Hub）；ECS 是运行容器的编排工具。 |
| **EC2 Launch Type** | **Fargate** | 选 EC2 为了完全控制或利用已有 RI；选 Fargate 为了省事（No Ops）。 |
| **Task Role** | **Task Execution Role** | Task Role 给代码用；Execution Role 给 ECS Agent 用（拉镜像/日志）。 |
| **Service Auto Scaling** | **Cluster Auto Scaling** | Service 扩 Task 数量；Cluster 扩 EC2 实例数量（当资源不足时）。 |
| **ECS** | **EKS** | ECS 是 AWS 原生，简单易用，深度集成；EKS 是标准 Kubernetes，适合开源生态迁移，略复杂。 |
| **EFS** | **EBS** | 多个容器（跨 AZ）共享存储必须用 **EFS**。Fargate 仅原生支持 EFS 挂载（EBS 支持有限且主要用于 EC2 模式）。 |

**最后的黄金法则：**
*   看到“**Serverless / No manage EC2**” -> 选 **Fargate**。
*   看到“**Legacy app migration / granular control**” -> 考虑 **EC2 模式**。
*   看到“**Privileges / Permissions**” -> 检查是 **Task Role** 还是 **Execution Role**。
*   看到“**Save Cost**” 且任务可中断 -> 选 **Spot 模式**。
