# Lustre 与 Ceph 核心组件详解

---

## 一、Lustre 核心组件

Lustre 是面向 HPC（高性能计算）的并行分布式文件系统，采用"管理-元数据-数据"三层分离架构。整体分为 4 类角色、6 个核心组件：MGS/MGT、MDS/MDT、OSS/OST、Client。

### 1. MGS（Management Server，管理服务器）

**MGS** 是 Lustre 文件系统的全局配置管理中心。

- **职责**：存储和管理整个 Lustre 文件系统的配置信息，包括所有节点信息、文件系统名称、网络拓扑等
- **MGT（Management Target）**：MGS 使用的后端存储设备，一般是一个磁盘分区
- **特点**：一个 MGS 可以管理多个 Lustre 文件系统；通常与 MDS 部署在同一节点上以节省资源
- **高可用**：支持主备模式，通过 `--servicenode` 参数指定故障转移节点

> 客户端挂载文件系统时，首先连接 MGS 获取配置信息，然后才能定位到 MDS 和 OSS。

### 2. MDT（Metadata Target，元数据目标）

**MDT** 是存储文件系统元数据的后端存储设备，由 MDS 管理。

- **职责**：存储文件的元数据，包括文件名、目录结构、权限、文件大小、时间戳、扩展属性（EA）等
- **MDS（Metadata Server）**：管理 MDT 的服务进程，对外提供元数据服务
- **文件布局（Layout）**：每个文件的 EA 中记录了该文件被分割为多少个对象，以及这些对象分布在哪些 OST 上。例如一个 12MB 的文件，条带大小为 4MB，则 EA 记录了 3 个对象分别在 OST p1、p2、p3 上
- **DNE（Distributed Namespace，分布式命名空间）**：Lustre 2.4+ 支持多个 MDT，通过 `--index` 区分，可将不同目录的元数据分布到不同 MDT 上，突破单 MDT 元数据性能瓶颈
- **数据路径**：元数据操作走 MDS/MDT，实际数据传输不经过 MDS，直接由客户端与 OSS 交互

### 3. OSS/OST（Object Storage Server/Target，对象存储服务器/目标）

**OSS** 是实际存储用户数据的前端服务，**OST** 是 OSS 管理的后端块设备。

- **OSS 职责**：对外暴露数据存储服务，接收客户端的数据读写请求，将数据写入后端 OST
- **OST 职责**：存储用户的文件数据对象（Object），每个 OST 对应一个块设备（如磁盘分区）
- **容量计算**：Lustre 文件系统总容量 = 所有 OST 容量之和
- **性能特点**：多个 OSS 可并行提供数据读写，聚合带宽随 OSS 数量线性增长；每个 OST 可达 128TB
- **软件栈**：OSS 包含 OBD Filter（将 Lustre 请求转换为后端文件系统请求）和 LDLM（分布式锁管理）模块
- **后端文件系统**：默认使用 ldiskfs（增强版 ext4），也支持 ZFS
- **高可用**：支持主备模式，通过 `--servicenode` 配置故障转移

### 4. Client（客户端）

**Client** 是挂载和使用 Lustre 文件系统的节点，通过标准 POSIX 接口访问。

- **MGC（Management Client）**：负责与 MGS 通信，获取文件系统配置信息
- **MDC（Metadata Client）**：负责与 MDS 通信，查询文件元数据和布局信息（EA）
- **OSC（Object Storage Client）**：负责与 OSS 通信，执行实际的数据读写操作
- **工作流程**：
  1. 客户端通过 VFS 发起文件操作请求（如 `read`）
  2. MDC 向 MDS 查询文件的 EA（获取文件对象在哪些 OST 上）
  3. OSC 根据 EA 信息，直接与对应的 OSS 建立连接，并行读写数据
- **通信协议**：Lustre 服务端与客户端之间通过 Portal RPC（LNET 之上）通信

### Lustre 组件总结

| 组件 | 角色 | 管理对象 | 存储内容 |
|------|------|---------|---------|
| MGS | 配置管理 | MGT | 文件系统全局配置 |
| MDS | 元数据服务 | MDT | 文件名、目录结构、权限、Layout |
| OSS | 数据存储服务 | OST | 文件数据对象（条带化存储） |
| Client | 终端访问 | — | 无持久化存储，通过 MGC/MDC/OSC 访问 |

---

## 二、Ceph 核心组件

Ceph 是统一的分布式存储系统，基于 RADOS（Reliable Autonomic Distributed Object Store）架构，同时提供对象存储、块存储和文件存储三种接口。核心组件包括：MON、MGR、OSD、MDS、RGW。

### 1. MON（Monitor，监视器）

**MON** 是 Ceph 集群的"大脑"，负责维护集群的全局状态和一致性。

- **职责**：
  - 维护集群地图（Cluster Map），包括 Monitor Map、OSD Map、PG Map、MDS Map、CRUSH Map
  - 管理集群成员关系和节点状态
  - 处理客户端认证和授权
- **共识机制**：使用 Paxos/Raft 协议在 MON 节点之间达成共识，确保集群状态的一致性
- **部署要求**：必须是奇数个（通常 3 或 5 个），通过多数派原则防止"脑裂"（Split-Brain）
- **不参与数据路径**：MON 不直接存储用户数据，也不参与 PG 的具体维护工作
- **工作流程**：客户端访问集群时，首先从 MON 获取最新的 Cluster Map，然后根据 CRUSH 算法直接计算数据存储位置，无需再经过 MON

### 2. MGR（Manager，管理器）

**MGR** 是 Ceph 的运维管理中枢，负责收集集群运行指标并提供管理接口。

- **职责**：
  - 收集集群的监控数据（存储利用率、网络状态、性能指标等）
  - 提供 Web Dashboard 可视化界面
  - 运行插件模块扩展功能（如 Prometheus 监控、Zabbix 集成、磁盘预测等）
  - 动态调整集群配置参数
- **高可用**：多个 MGR 节点之间通过选举产生一个 Active 主节点，其余为 Standby。主节点故障时自动切换
- **与 MON 的区别**：MON 负责集群状态一致性，MGR 负责运维数据的收集和展示
- **模块化设计**：MGR 支持多种插件，可根据需求启用不同功能模块

### 3. OSD（Object Storage Daemon，对象存储守护进程）

**OSD** 是 Ceph 中最核心的存储组件，实际负责数据的存储、复制、恢复和再平衡。

- **职责**：
  - **数据存储**：将数据对象写入物理磁盘，并确保持久性和完整性
  - **数据读写**：处理客户端的数据读写请求
  - **数据复制**：根据 Pool 的副本数配置，将数据复制到其他 OSD
  - **数据恢复**：当某个 OSD 故障时，其他 OSD 自动重建丢失的数据副本
  - **数据再平衡**：当 OSD 加入或离开集群时，自动重新分布数据
- **实现方式**：每个 OSD 是一个独立进程，通常一块物理磁盘对应一个 OSD 进程
- **CRUSH 算法**：OSD 配合 CRUSH 算法决定数据对象的存储位置，无需中心化调度
- **心跳检测**：OSD 之间定期进行心跳检测，及时发现故障并触发恢复
- **性能优化**：支持内存缓存加速、并行 I/O 处理、可配置的 I/O 调度策略

#### OSD 写入流程

1. 客户端从 MON 获取 Cluster Map
2. 客户端根据对象 ID 计算 PG（Placement Group）
3. 根据 CRUSH 算法找到该 PG 的主 OSD
4. 客户端将数据发送给主 OSD
5. 主 OSD 写入本地存储，同时将数据同步到副本 OSD
6. 所有副本 OSD 确认写入完成后，主 OSD 向客户端返回成功

### 4. MDS（Metadata Server，元数据服务器）

**MDS** 是 Ceph 文件系统（CephFS）的元数据管理组件。

- **职责**：
  - 管理 CephFS 的目录结构和文件元数据（文件名、权限、大小等）
  - 快速响应目录遍历和元数据查询请求
  - 确保文件系统元数据的一致性
- **使用场景**：仅在 CephFS（文件存储）中使用，块存储（RBD）和对象存储（RGW）不需要 MDS
- **多 MDS**：支持部署多个 MDS 节点，提高元数据处理的性能和可靠性
  - 可配置 Active-Standby 模式（一个活跃，其余热备）
  - 也可配置多 Active 模式，将不同目录的元数据负载分发到不同 MDS
- **注意**：MDS 管理的元数据本身也存储在 RADOS 中（通过 OSD），MDS 只负责缓存和加速元数据访问

### 5. RGW（RADOS Gateway，RADOS 网关）

**RGW** 是 Ceph 的对象存储网关，提供兼容 S3 和 Swift 的 HTTP RESTful API 接口。

- **职责**：
  - 将 HTTP 请求转换为 RADOS 对象操作
  - 提供兼容 Amazon S3 和 OpenStack Swift 的 API
  - 管理用户认证、访问控制（ACL）、多租户隔离
  - 支持多站点同步（Multi-Site Replication）
- **使用场景**：构建云存储服务，如图片存储、视频存储、静态网站托管、数据湖等
- **部署方式**：通常与负载均衡器（如 HAProxy）配合，部署多个 RGW 实例实现高可用
- **数据存储**：RGW 将对象数据存入 RADOS Pool，元数据也保存在 RADOS 中，无需额外元数据服务

### Ceph 组件总结

| 组件 | 角色 | 是否参与数据路径 | 存储内容 |
|------|------|:---:|---------|
| MON | 集群状态维护 | 否 | Cluster Map、节点状态 |
| MGR | 监控运维管理 | 否 | 性能指标、Dashboard |
| OSD | 实际数据存储 | 是 | 用户数据对象 |
| MDS | 文件系统元数据 | 否（仅 CephFS） | 目录结构、文件属性 |
| RGW | 对象存储网关 | 是 | S3/Swift 对象数据 |

---

## 三、Lustre 与 Ceph 组件对比

| 维度 | Lustre | Ceph |
|------|--------|------|
| **配置管理** | MGS（独立配置管理服务） | MON（集群状态 + 配置，一体） |
| **元数据** | MDS + MDT（独立元数据节点） | MDS（仅 CephFS 需要，元数据存在 RADOS 中） |
| **数据存储** | OSS + OST（专用数据节点） | OSD（通用数据守护进程） |
| **运维管理** | 无独立组件，靠命令行 | MGR（Dashboard、监控、插件） |
| **对象网关** | 无（纯文件系统） | RGW（S3/Swift 兼容） |
| **客户端** | Lustre Client（内核模块） | librados / RBD / CephFS / RGW |
| **数据分布** | 文件级条带化（Layout EA） | 对象级 CRUSH 算法 |
| **副本/冗余** | 无内置副本（靠 RAID 或外部冗余） | 内置多副本 / 纠删码 |
| **故障恢复** | 手动或 HA 故障转移 | 自动检测 + 自动恢复 + 自动再平衡 |

---

## 四、典型工作流对比

### Lustre 读文件流程

```
1. Client 发起 read() 系统调用
2. MDC → MDS：查询文件 Layout（EA），获取文件对象所在的 OST 列表
3. OSC → 多个 OSS：并行读取各 OST 上的文件对象数据
4. Client 组装数据返回给应用
```

### Ceph 写入对象流程

```
1. Client 从 MON 获取 Cluster Map
2. Client 计算对象 ID → PG → CRUSH 定位主 OSD
3. Client → 主 OSD：发送写入请求
4. 主 OSD → 副本 OSD：同步写入数据
5. 所有副本确认 → 返回客户端成功
```

---

## 五、关键差异总结

1. **Lustre 有独立配置中心 MGS**，Ceph 的配置管理整合在 MON 中
2. **Lustre 元数据直接存在本地 MDT 磁盘上**，Ceph 的 MDS 元数据最终也存储在 RADOS（OSD）中
3. **Lustre 无内置副本机制**，依赖底层 RAID 或存储冗余；Ceph 内置多副本/纠删码，自动故障恢复
4. **Ceph 多了 MGR 和 RGW**，提供了更强的运维管理和对象存储能力
5. **Lustre 客户端是内核模块**，提供 POSIX 文件系统接口；Ceph 提供 librados、RBD、CephFS、RGW 多种接入方式
6. **Lustre 数据路径全并行**（Client 直连 OSS），带宽近线性扩展；Ceph 有 PG 同步和副本写入开销