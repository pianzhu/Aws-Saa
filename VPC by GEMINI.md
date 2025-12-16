# Amazon VPC SAA Deep Dive

本文档旨在为 AWS Solutions Architect Associate (SAA) 考生提供 Amazon VPC 的深度解析。内容涵盖架构设计、成本优化、高可用性、安全性及性能优化，并针对易混淆概念进行对比分析。

---

## 1. Amazon VPC 基础与子网设计

### 1.1 VPC 与 CIDR 规划
*   **核心概念**：VPC 是 AWS 云中的逻辑隔离虚拟网络。创建 VPC 时必须指定 IPv4 CIDR 块（如 `10.0.0.0/16`）。
*   **CIDR 大小**：最小 `/28` (16 IP)，最大 `/16` (65,536 IP)。
*   **CIDR 限制**：
    *   创建后主 CIDR 不可修改，但可以添加辅助 CIDR。
    *   CIDR 块不能与其他连接的网络（本地 IDC、Peering 的 VPC）重叠。
*   **保留地址**：每个子网有 5 个 IP 被 AWS 保留：
    *   `.0` (网络地址)
    *   `.1` (VPC 路由器)
    *   `.2` (DNS 服务器)
    *   `.3` (保留供将来使用)
    *   `.255` (广播地址，AWS VPC 不支持广播)

### 1.2 子网类型 (Subnet Types)
*   **公有子网 (Public Subnet)**：
    *   路由表中包含指向 **IGW (Internet Gateway)** 的路由 (`0.0.0.0/0 -> IGW`)。
    *   实例拥有公有 IP (或弹性 IP) 可直接访问互联网。
    *   **典型用途**：负载均衡器 (ALB)、Bastion Host、NAT Gateway。
*   **私有子网 (Private Subnet)**：
    *   路由表中**没有**指向 IGW 的路由。
    *   通过 **NAT Gateway** 或 **NAT Instance** 访问互联网（单向：出站请求响应，拒绝入站）。
    *   **典型用途**：应用服务器、后端服务、数据库。
*   **隔离子网 (Isolated Subnet)** (有时也称为纯私有子网)：
    *   既没有 IGW 也没有 NAT 路由。
    *   只能访问 VPC 内部资源或通过 VPC Endpoints 访问 AWS 服务。
    *   **典型用途**：高度敏感的数据库、不需要互联网补丁的内部系统。

---

## 2. 路由与网关组件 (Routing & Gateways)

### 2.1 路由表 (Route Tables)
*   **主路由表 (Main Route Table)**：VPC 自动创建，所有未显式关联子网的默认路由表。**最佳实践**：主路由表仅用于私有子网，显式创建自定义路由表关联公有子网。
*   **路由优先级**：最长前缀匹配 (Longest Prefix Match) 优先。

### 2.2 互联网网关 (IGW)
*   **特性**：水平扩展、高可用、无带宽限制组件。
*   **配置**：Attach 到 VPC -> 在路由表中添加 `0.0.0.0/0 -> igw-xxx`。
*   **限制**：一个 VPC 只能附加一个 IGW。

### 2.3 NAT Gateway vs NAT Instance (SAA 高频考点)

| 特性 | NAT Gateway (托管服务) | NAT Instance (EC2 实例) |
| :--- | :--- | :--- |
| **高可用性** | **高**。AWS 托管，自动在可用区(AZ)内冗余。 | **低**。单点故障，需通过 Auto Scaling 脚本实现故障转移。 |
| **带宽** | 自动扩展 (高达 45 Gbps)。 | 取决于 EC2 实例类型和网络性能。 |
| **维护** | 零维护 (AWS 管理补丁)。 | 需用户管理 OS 补丁、软件更新。 |
| **配置** | 简单。需分配一个 EIP。 | 复杂。需禁用 **"Source/Destination Check"**。 |
| **成本** | 按小时 + 数据处理费 (较贵)。 | 按实例小时计费 (通常较便宜，适合小流量)。 |
| **端口转发** | 不支持。 | 支持 (可兼作 Bastion Host)。 |

> **SAA 关键策略**：
> *   **首选 NAT Gateway** 用于生产环境（HA 和性能）。
> *   **NAT Gateway 是 AZ 级别的资源**。为了实现多 AZ 高可用，必须在**每个 AZ** 创建一个 NAT Gateway，并让该 AZ 的私有子网指向本地的 NAT Gateway (避免跨 AZ 流量费用和单点故障)。

### 2.4 仅出口互联网网关 (Egress-Only Internet Gateway)
*   **适用场景**：仅用于 **IPv6**。
*   **功能**：允许 VPC内 IPv6 实例出站访问互联网，但阻止互联网发起的入站连接（类似于 IPv4 的 NAT Gateway 也就是有状态防火墙的功能，但它是针对 IPv6 的网关）。

---

## 3. 多 VPC 连接与混合网络

### 3.1 VPC 对等连接 (VPC Peering)
*   **功能**：两个 VPC 之间的私有网络连接 (可跨账户、跨区域)。
*   **限制 (非传递性)**：
    *   如果 A 对等 B，B 对等 C，A **不能** 通过 B 访问 C。必须创建 A-C 对等连接。
    *   CIDR 不能重叠。
*   **性能**：如同同一网络，带宽无瓶颈。

### 3.2 Transit Gateway (TGW) - 集中式网络枢纽
*   **架构**：Hub-and-Spoke 模型，解决复杂的网状 Peering 问题。
*   **功能**：连接数千个 VPC 和本地网络 (VPN/Direct Connect)。
*   **传递性**：支持传递路由 (A -> TGW -> B)。
*   **优势**：简化管理，集中监控。
*   **组播**：支持 IP Multicast (Peering 不支持)。

### 3.3 VPC 共享 (VPC Sharing)
*   **机制**：基于 AWS Resource Access Manager (RAM)。
*   **场景**：同属一个 AWS Organization。Owner 账户创建 VPC 和子网，分享给 Participant 账户。
*   **优势**：
    *   **资源隔离，网络统一**：不同账户的资源(EC2/RDS)在同一子网内，可通过 private IP 通信。
    *   **避免 Peering/TGW 费用**：同一 VPC 内通信免费(同AZ)或低费(跨AZ)。
    *   **职责分离**：网络团队管理 VPC，应用团队管理实例。

---

## 4. 服务端点 (VPC Endpoints) - 安全访问 AWS 服务

无需经过 IGW/NAT，流量保留在 AWS 骨干网。

### 4.1 Gateway Endpoints
*   **支持服务**：**仅 S3 和 DynamoDB**。
*   **原理**：修改路由表 (Route Table)，添加指向 Gateway Endpoint 的路由 (Prefix List)。
*   **成本**：**免费**。
*   **限制**：只能在 VPC 内部访问，不支持跨 Region，不支持从本地通过 VPN/DX 访问。

### 4.2 Interface Endpoints (基于 PrivateLink)
*   **支持服务**：大多数 AWS 服务 (SQS, SNS, Kinesis, EC2 API, etc.)。
*   **原理**：在子网中创建一个 **ENI (Elastic Network Interface)**，提供私有 IP。
*   **成本**：按小时 + 数据处理费。
*   **优势**：支持从本地网络 (VPN/DX) 访问，支持跨 Region Peering 访问。

---

## 5. 安全性：NACL vs Security Group

| 特性 | Network ACL (NACL) | Security Group (SG) |
| :--- | :--- | :--- |
| **作用范围** | **子网级别** (Subnet Level)。 | **实例级别** (Instance Level: EC2, RDS, ENI)。 |
| **状态** | **无状态 (Stateless)**。入站和出站流量需分别配置。 | **有状态 (Stateful)**。允许入站即自动允许响应出站。 |
| **规则类型** | 允许 (Allow) 和 **拒绝 (Deny)**。 | 仅 **允许 (Allow)**。默认全部拒绝。 |
| **规则顺序** | 按数字顺序评估 (如 100, 200)，匹配即停止。 | 评估所有规则，合并判断 (OR 逻辑)。 |
| **典型用途** | 额外的安全层，阻止特定 IP (黑名单)。 | 主要的防火墙，定义应用访问规则 (白名单)。 |

> **SAA 场景**：若题目要求“阻止某个特定恶意 IP”，必须使用 **NACL** (SG 无法显式 Deny)。

### 5.1 AWS Network Firewall
*   **定位**：托管的高级防火墙服务，用于 VPC 边界。
*   **功能**：
    *   第 3-7 层保护。
    *   状态检测 (Stateful inspection)。
    *   **入侵防御系统 (IPS)**：检查流量内容 (如 SQL 注入特征)。
    *   URL 过滤 (FQDN filtering)。
*   **对比**：比 NACL/SG 更强大 (NACL/SG 仅基于 IP/Port，Network Firewall 可通过域名过滤)。

---

## 6. 工具与故障排除

### 6.1 VPC Flow Logs
*   **功能**：捕获进出 ENI 的 IP 流量信息 (元数据，不包含报文内容)。
*   **存储**：发送到 CloudWatch Logs 或 S3。
*   **用途**：
    *   故障排除 (为什么连接超时？)。
    *   安全审计 (发现异常 IP 扫描)。
*   **状态码**：`ACCEPT` vs `REJECT`。

### 6.2 Reachability Analyzer
*   **功能**：**静态配置分析**工具。
*   **用途**：诊断“为什么 A 连不上 B？”。
*   **原理**：不发送实际数据包，而是分析路由表、SG、NACL 等配置，给出“可达”或“不可达及原因”的报告。

### 6.3 Network Access Analyzer
*   **功能**：帮助识别非预期的网络访问路径。
*   **用途**：合规性检查 (例如：验证私有子网是否真的无法从互联网访问)。

### 6.4 弹性网络接口 (ENI)
*   **概念**：VPC 中的虚拟网卡。
*   **特性**：
    *   主 ENI (eth0) 随实例创建和销毁。
    *   辅助 ENI 可以独立创建，并在实例间迁移 (用于故障转移 HA)。
    *   可以有多个私有 IP，一个公有/EIP。

---

## 7. 架构考点总结 (Architectural Pillars)

### 7.1 成本优化 (Cost Optimization)
*   **NAT Gateway**：如果流量巨大，NAT Gateway 处理费可能很高。可考虑使用 VPC Endpoints (Gateway 类型免费，Interface 类型可能更便宜且安全) 直接访问 AWS 服务。
*   **跨 AZ 流量**：尽量保持流量在同 AZ 内。跨 AZ 通信收费。
*   **Release Unused EIP**：未绑定或绑定但停止的实例上的 EIP 会收费。

### 7.2 高可用与容错 (High Availability & Fault Tolerance)
*   **多 AZ 部署**：子网、NAT Gateway、ALB 必须跨多个 AZ 部署。
*   **Site-to-Site VPN**：使用两条隧道实现冗余；或 VPN + Direct Connect 作为备份。

### 7.3 安全最佳实践 (Security)
*   **纵深防御**：同时使用 SG (最小权限) 和 NACL (子网边界防护)。
*   **私有化**：数据库和后端服务绝不直接暴露在公有子网，始终使用 Bastion 或 SSM Session Manager 访问。
*   **VPC Endpoints**：避免流量穿越公网访问 S3/DynamoDB。

### 7.4 性能优化 (Performance)
*   **Jumbo Frames**：支持 MTU 9001，提高吞吐量 (需实例间都在 VPC 内)。
*   **Enhanced Networking (ENA)**：高性能实例使用 ENA 获取更高带宽和更低延迟。
