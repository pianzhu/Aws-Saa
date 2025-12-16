# AWS SAA 深度剖析：Elastic Beanstalk 与存储服务

> **文档说明**：本文档专为 AWS Certified Solutions Architect - Associate (SAA) 备考设计。内容紧扣 SAA 考试的核心架构支柱：**成本优化 (Cost Optimization)**、**高可用性 (High Availability)**、**弹性 (Elasticity)** 和 **安全性 (Security)**。针对考试中常见的易混淆概念进行了对比分析。

---

## 第一部分：AWS Elastic Beanstalk

Elastic Beanstalk (EB) 是 SAA 考试中关于**计算服务部署**和**自动化运维**的重点。考试通常不会考察底层代码怎么写，而是考察**如何选择正确的部署策略**以满足业务对“停机时间”、“成本”和“回滚能力”的要求，以及**如何架构解耦**。

### 1. 部署策略 (Deployment Policies) —— 考点核心

这是 EB 最常考的内容。你需要根据场景关键词选择策略。

| 策略名称 | 行为描述 | 停机时间 | 对生产环境影响 | 成本 | 回滚速度 | 适用场景关键词 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **All at once** (一次全部) | 同时在所有实例上部署新版本及重启。 | **有** (数分钟不可用) | 服务全断 | **最低** | 慢 (需重新部署旧版) | 开发环境、不介意停机的低成本场景。 |
| **Rolling** (滚动) | 分批次更新（如每次 30%）。旧版下线，新版上线。 | **无** (但容量降低) | 性能可能下降 (容量减少) | 低 | 慢 (需手动滚动回旧版) | 生产环境，稍微降低容量可接受，无需额外成本。 |
| **Rolling with additional batch** (附加批次滚动) | 先启动一批新实例，然后滚动更新旧实例。始终保持满容量。 | **无** | 性能无影响 (维持 100% 容量) | 中 (短时间多出一批实例) | 慢 | 生产环境，**不能**降低服务容量，但不想花费由蓝绿部署带来的双倍成本。 |
| **Immutable** (不可变) | 创建一个新的 ASG (Auto Scaling Group) 部署新版，通过后切换流量，销毁旧 ASG。 | **无** | 性能无影响 | 高 (部署期间双倍容量) | **快** (直接切回旧 ASG) | 需要**快速回滚**机制；确保新环境纯净，避免配置漂移。 |
| **Blue/Green** (蓝/绿) | *注意：这不是 EB 的原生配置选项，而是通过建立新环境实现的架构模式。* 创建完全独立的新环境(Green)，通过 **Route 53** 或 **CNAME Swap** 切换流量。 | **无** (瞬间切换) | 性能无影响 | **最高** (双倍环境) | **最快 (即时)** | 业务极其关键，零停机，需**即时**回滚，测试完整环境后再切换。 |

> **关键对比分析**：
> *   **Immutable vs Blue/Green**: 
>     *   **Immutable** 是在**同一个环境内**创建一个临时的 ASG，部署完后旧 ASG 被销毁，最终只剩一个 ASG。
>     *   **Blue/Green** 是创建**两个完全独立的环境 (Environment)**，通过 DNS/Swap URL 切换。Blue/Green 比 Immutable 更彻底，回滚最快（只是切个 DNS），但成本最高。
> *   **Rolling vs Rolling with batch**: 区别在于是否容忍部署期间容量（性能）下降。题目如果说 "Must maintain full capacity" (必须维持满容量)，排除 "Rolling"。

### 2. 环境配置：Web Server Tier vs Worker Tier

考试常考**架构解耦 (Decoupling)** 和**处理长耗时任务**。

*   **Web Server Tier (Web 服务器环境)**:
    *   用于处理标准的 HTTP 请求。
    *   通常架构：Load Balancer + Auto Scaling Group。
    *   **考点**：如果这个环境里的应用需要处理一个耗时 10 分钟生成报表的任务，会导致 HTTP 连接超时或 Web Server 堵塞。怎么优化？ —— **拆分！**
*   **Worker Tier (工作线程环境)**:
    *   用于后台处理长耗时任务。
    *   核心组件：**SQS Queue** (由 EB 自动创建) + Auto Scaling Group + **SQS Daemon** (运行在实例上的守护进程)。
    *   **工作流**：Web Tier 将任务推送到 SQS -> Worker Tier 的 Daemon 从 SQS 读取消息 -> POST 到本地 localhost 应用进行处理。
    *   **SAA 经典答案**：遇到“Web 应用因高负载处理任务变慢”或“生成视频/报表导致超时”，选 **"Offload tasks to SQS and create a Worker Tier to process them"**。

### 3. 配置管理 (.ebextensions)

*   **位置**：必须位于项目根目录的 `.ebextensions/` 文件夹下。
*   **格式**：YAML 或 JSON。
*   **作用**：安装软件包 (yum install)、创建文件、运行 shell 命令、配置负载均衡器参数等。
*   **考点**：所有的环境级别配置（如修改 Apache/Nginx 配置、安装依赖库）都应代码化放入 `.ebextensions`，以保证环境的**可重现性**和**不可变性**。

### 4. 平台更新 (Managed Platform Updates)

*   **功能**：AWS 自动修补操作系统、语言运行时（Java, Python, Node.js 等）和应用服务器的版本。
*   **机制**：使用 **Immutable** 部署机制（即使你平时配置的是 Rolling）。它会创建新实例安装新补丁，健康检查通过后替换旧实例，确保更新不会搞挂现有环境。

---

## 第二部分：AWS 存储服务 (Storage Services)

SAA 考试中，存储主要考察 S3, EFS, EBS (在计算部分涉及), FSx 和 Storage Gateway。诀窍是根据**访问模式、协议、操作系统**来选择。

### 1. Amazon S3 (Simple Storage Service) —— 对象存储

*   **特点**：无限容量，HTTP/HTTPS 访问，Region 级别（跨 AZ 高可用）。
*   **存储类别 (Storage Classes) —— 成本优化考点**：
    *   **S3 Standard**: 频繁访问，毫秒级延迟。最贵。
    *   **S3 Intelligent-Tiering**: **只有这个**能自动根据访问模式在层级间移动（且无取回费）。**考点**：题目说“访问模式不可预测 (Unpredictable) 或一直在变”，选它。
    *   **S3 Standard-IA (Infrequent Access)**: 访问频率低，但需要**毫秒级**访问。有**取回费**。最少存储 30 天。
    *   **S3 One Zone-IA**: 数据可重建（非关键数据），便宜 20%。单 AZ，无容灾。
    *   **S3 Glacier Instant Retrieval**: 归档数据，但需要**毫秒级**读取。
    *   **S3 Glacier Flexible Retrieval**: 归档，取回需几分钟到几小时。便宜。
    *   **S3 Glacier Deep Archive**: 最便宜，取回需 12-48 小时。**考点**：法规要求保留数据 7-10 年，且几乎不访问。
*   **生命周期策略 (Lifecycle Policies)**：自动化在上述层级间转移数据以优化成本。
*   **安全性**：
    *   **S3 Bucket Policies**: 资源级权限，允许跨账户访问，控制 Public 通用设置。
    *   **S3 ACLs**: 旧功能，通常建议禁用（使用 Block Public Access），除非题目明确提及特殊的单对象权限场景。
    *   **Block Public Access**: 账户/存储桶级别的安全开关，防止意外数据泄露。**默认开启**。
    *   **Encryption**: Server-Side (SSE-S3, SSE-KMS, SSE-C) 和 Client-Side。SSE-KMS 主要考点是 KMS 的 API 限制（如果上传很多小文件可能会触发 KMS Quota）。
*   **性能**：
    *   **Multipart Upload**: 文件 > 100MB 建议用，> 5GB 必须用。提高上传速度和成功率。
    *   **S3 Transfer Acceleration**: 利用 CloudFront 边缘节点加速全球上传。
    *   **S3 Byte-Range Fetches**: 并行下载部分数据，加速下载。

### 2. Amazon EFS (Elastic File System) —— Linux 文件共享

*   **协议**：NFSv4。
*   **兼容性**：仅限 **Linux** 实例 (POSIX 兼容)。**Windows 不行**。
*   **架构**：跨多个 AZ 存储，高可用。可以被成百上千个 EC2 **并发挂载**。
*   **模式**：
    *   **Performance Mode**: General Purpose (默认) vs Max I/O (大数据分析，高并发)。
    *   **Throughput Mode**: Bursting vs Provisioned vs Elastic。
*   **存储类**：也有 Standard 和 IA (Infrequent Access)，可配置生命周期自动转入 EFS IA 节省成本。
*   **关键考点**：
    *   如果题目提到：**“多个 Linux 实例需要同时读写同一个共享文件目录”** -> **EFS**。
    *   如果是 **Web Server 集群共享静态内容** 或 **WordPress 媒体库** -> **EFS** 也是这里的一个可能选项（虽然 S3 + CloudFront 往往更好，但如果是遗留应用需要文件系统接口，则选 EFS）。
    *   **Lambda 集成**：Lambda 函数可以挂载 EFS 来处理大文件或共享状态。

### 3. Amazon FSx 系列 —— 专用文件系统

*   **FSx for Windows File Server**:
    *   **协议**：**SMB**。
    *   **兼容性**：原生 **Windows** 文件系统。支持 Active Directory (AD) 集成。
    *   **考点**：**“多个 Windows 实例共享存储”**、**“SMB 协议”**、**“迁移现有的 Windows 文件服务器”** -> **FSx for Windows**。千万别选 EFS (EFS 只支持 Linux)。
*   **FSx for Lustre**:
    *   **场景**：高性能计算 (HPC)、机器学习、视频渲染。
    *   **特点**：极高的吞吐量 (Throughput) 和 IOPS。可以与 S3 **无缝链接** (Linked to S3 bucket)，把 S3 当作这个高性能文件系统的后端“磁带库”使用。
    *   **考点**：题目出现 “HPC”、“Parallel file system”、“Millions of IOPS” 或 “Lustre” -> **FSx for Lustre**。

### 4. AWS Storage Gateway —— 混合云存储

连接本地数据中心 (On-Premises) 和 AWS 云存储。

| 类型 | 用途 | 关键词/场景 |
| :--- | :--- | :--- |
| **S3 File Gateway** | 让本地服务器通过 **NFS/SMB** 协议访问 S3 对象。 | 混合云，本地应用像读写对地文件一样读写 S3。数据最终存在 S3。 |
| **Volume Gateway (Stored Mode)** | 数据全存在本地，**异步备份**到 AWS EBS 快照。 | 需要**超低延迟**访问完整数据集，同时利用 AWS 做灾备。 |
| **Volume Gateway (Cached Mode)** | 数据存在 AWS S3，本地只缓存**热数据** (Most recently accessed)。 | 本地存储**空间有限**，但希望扩展存储到云端且保持常用数据低延迟。 |
| **Tape Gateway** | 虚拟磁带库 (VTL)。 | 替代物理磁带备份，使用 **Glacier** 长期归档，无需改变现有的备份工作流 (如 Veeam, NetBackup)。 |

---

## 第三部分：存储服务对比与决策矩阵 (Cheat Sheet)

| 场景需求 | 推荐服务 | 为什么？/ 排除项 |
| :--- | :--- | :--- |
| **静态网站托管**，最低成本 | **S3** | 开启 Static Website Hosting (成本比 EC2/EB 低太多)。 |
| **EC2 Linux** 实例间**共享**文件系统 | **EFS** | EBS 只能单挂载（io1/2 虽可多挂载但有限制且不是文件系统）；S3 不是 POSIX 文件系统。 |
| **EC2 Windows** 实例间**共享**文件系统 | **FSx for Windows** | EFS 不支持 Windows。S3 不支持 SMB。 |
| **HPC** 高性能计算，并行处理 | **FSx for Lustre** | 专为高性能设计。 |
| 本地机房扩容，想用云存储替代**磁带库** | **Tape Gateway** | 关键词 Backup, Archive, Tape replacement。 |
| 需要块存储 (Block Storage)，高性能数据库 (Oracle/SQL Server) | **EBS** (io2/gp3) | S3/EFS 是文件/对象存储，不适合运行高性能数据库。 |
| **跨区域 (Cross-Region)** 数据复制 | **S3 CRR** | 用于合规性或降低异地访问延迟。 |
| 数据量巨大 (PB级)，网络带宽不够上传 | **AWS Snow Family** | Snowball Edge (迁移 + 计算), Snowmobile (Exabytes)。 |

### 常见误区澄清

1.  **S3 Transfer Acceleration vs CloudFront**：
    *   Transfer Acceleration 是为了**加速上传**到 S3 bucket。
    *   CloudFront 是为了**加速下载/分发**内容给全球用户。
2.  **EBS vs EFS 性能**：
    *   一般单机 IO 性能：EBS (io2 Block Express) > EFS。
    *   EFS 的优势在于**吞吐量**随容量扩展，以及**分布式并发访问**。
3.  **Instance Store (临时存储) vs EBS**：
    *   Instance Store：物理连接在主机上，**极高 IOPS**，但是**临时**的（关机数据丢失）。适用于缓存、临时处理缓冲区。
    *   题目如果说“数据不能丢失” -> 绝对不能只用 Instance Store。

---
**复习建议**：不要死记硬背参数（如最大文件大小），重点关注**使用场景 (Use Cases)** 和 **成本/性能权衡 (Trade-offs)**。祝考试顺利！
