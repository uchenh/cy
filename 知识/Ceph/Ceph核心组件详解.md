# Ceph 核心组件详解

---

## 一、Ceph 架构概览

Ceph 是一个开源的、统一的分布式存储系统，设计用于提供优秀的性能、可靠性和可扩展性。Ceph 的架构分为三层：

| 层级 | 说明 | 组件 |
|------|------|------|
| **RADOS 层** | Reliable Autonomic Distributed Object Store，可靠自动化分布式对象存储 | MON、MGR、OSD |
| **RBD 层** | RADOS Block Device，块设备抽象层 | librados、RBD |
| **应用接口层** | 面向客户端的接入接口 | CephFS、RGW、iSCSI Gateway |

整个系统基于 CRUSH（Controlled Replication Under Scalable Hashing）算法实现数据分布，无需中心化调度。

---

## 二、MON（Monitor，监视器）

### 2.1 职责

MON 是 Ceph 集群的"大脑"，负责维护集群的全局状态和一致性。

**核心职责**：

- **维护集群地图（Cluster Map）**：包括以下关键映射
  - **MON Map**：监视器节点列表和状态
  - **OSD Map**：所有 OSD 的列表、状态、权重
  - **PG Map**：所有 PG 的归属和状态
  - **MDS Map**：MDS 节点列表和状态（CephFS 需要）
  - **CRUSH Map**：数据分布规则和故障域层次结构
- **管理集群成员关系**：监控节点上线/下线，维护集群拓扑
- **处理客户端认证**：验证客户端身份，授权访问
- **集群日志（Monlog）**：维护集群操作日志，确保状态变更可追溯

### 2.2 共识机制

Ceph MON 节点之间通过共识协议达成一致，确保集群状态的一致性：

- **协议**：使用 Paxos（旧版）或 Raft（新版 Octopus+）共识算法
- **多数派原则**：只要超过半数 MON 在线，集群就能正常运行
- **奇数部署**：通常部署 3 或 5 个 MON 节点，防止"脑裂"（Split-Brain）
  - 3 个 MON：最多容忍 1 个节点故障
  - 5 个 MON：最多容忍 2 个节点故障
  - 7 个 MON：最多容忍 3 个节点故障

### 2.3 选举机制

- **无固定主节点**：所有 MON 节点对等，共同维护集群状态
- **动态 Leader 选举**：在更新 Cluster Map 等操作时，动态选举一个 Leader 协调操作
- **故障切换**：Leader 失效时，剩余 MON 自动重新选举，保证服务连续性

### 2.4 与数据路径的关系

MON **不直接参与数据存储和读写**，它只负责状态维护：

```
客户端 → MON（获取 Cluster Map）→ 根据 CRUSH 算法直接计算 PG 位置 → 直接读写 OSD（不经过 MON）
```

客户端在首次连接时从 MON 获取 Cluster Map 的完整副本，后续的数据读写直接指向 OSD，无需再经过 MON。这意味着 MON 是单点瓶颈风险最低的设计——只要客户端持有有效 Cluster Map，即使 MON 暂时不可用，已有连接不会中断。

### 2.5 MON 状态管理命令

```bash
# 查看 MON 状态
ceph mon stat
ceph mon dump

# 查看单个 MON 状态
ceph daemon mon.{name} mon_status

# 列出所有 MON
ceph mon dump
```

---

## 三、MGR（Manager，管理器）

### 3.1 职责

MGR 是 Ceph 的运维管理中枢，负责收集集群运行指标并提供管理接口。

**核心职责**：

- **监控数据采集**：收集集群的存储利用率、网络状态、性能指标（IOPS、吞吐量、延迟）
- **Dashboard 提供**：提供 Web 可视化界面，实时展示集群健康状态、容量使用、性能趋势
- **插件模块运行**：支持多种插件扩展功能
  - **Prometheus**：导出监控指标供 Prometheus 采集
  - **Zabbix**：集成 Zabbix 监控系统
  - **Dokomo Tools**：磁盘健康预测（SMART 数据分析）
  - **Telemetry**：匿名使用数据上报（可关闭）
  - **Restful API**：提供 RESTful API 管理集群
- **配置管理**：动态调整集群配置参数，部分参数无需重启
- **PG 平衡器**：监控 PG 分布是否均衡，在 OSD 故障后辅助数据再平衡

### 3.2 高可用设计

- **Active-Standby 模式**：多个 MGR 节点中，只有一个 Active 主节点工作，其余 Standby
- **自动切换**：主 MGR 故障时，Standby 自动选举接管，无需人工干预
- **与 MON 的区别**：MGR 不保证集群状态一致性，仅提供管理和监控服务

### 3.3 模块系统

MGR 采用模块化设计，每个插件是一个独立模块，按需启用：

| 模块名称 | 功能 |
|---------|------|
| `dashboard` | Web Dashboard |
| `prometheus` | Prometheus 指标导出 |
| `restful` | RESTful API |
| `telemetry` | 遥测数据上报 |
| `localpool` | OSD 本地容量分析 |
| `crash` | 崩溃日志收集 |

### 3.4 常用 MGR 命令

```bash
# 查看 MGR 状态
ceph mgr stat
ceph mgr dump

# 查看 MGR 模块列表
ceph mgr module ls

# 启停模块
ceph mgr module disable dashboard
ceph mgr module enable dashboard
```

---

## 四、OSD（Object Storage Daemon，对象存储守护进程）

### 4.1 基本概念

**OSD** 是 Ceph 中最核心的存储组件，每个 OSD 是一个独立进程，通常一块物理磁盘对应一个 OSD 进程。

**数据对象**：Ceph 中的数据以对象（Object）形式存储，每个对象包含：
- **数据**：用户实际写入的内容
- **元数据**：对象 ID、大小、版本号、时间戳

### 4.2 核心职责

| 职责 | 说明 |
|------|------|
| **数据存储** | 将数据对象写入物理磁盘，确保持久性和完整性 |
| **数据读写** | 处理客户端的数据读写请求 |
| **数据复制** | 根据 Pool 的副本数配置，将数据复制到其他 OSD |
| **数据恢复** | OSD 故障时，其他 OSD 自动重建丢失的数据副本 |
| **数据再平衡** | 新 OSD 加入或离开时，自动重新分布数据 |
| **心跳检测** | 向 MON 报告状态，检测其他 OSD 健康情况 |
| **数据校验** | 定期校验数据完整性，发现并修复不一致 |

### 4.3 OSD 的数据存储结构

Ceph OSD 使用两个存储设备（可选 NVMe 优化）：

| 设备 | 用途 | 说明 |
|------|------|------|
| **WAL（Write-Ahead Log）** | 写前日志 | 所有写操作先写入 WAL，默认放在 SSD/NVMe 上，降低写延迟 |
| **DB（RocksDB）** | 元数据存储 | 存储 OSD 的元数据（对象 ID → 位置映射），默认放在 SSD/NVMe 上 |
| **DATA（磁盘/SSD）** | 实际数据存储 | 存放用户数据对象，可使用 HDD 或 SSD |

> 高性能配置：使用 NVMe SSD 同时作为 WAL 和 DB，HDD 作为 DATA，可以在控制成本的同时获得接近全闪存的写入性能。

### 4.4 PG（Placement Group，归置组）

**PG 是 OSD 之间的数据管理中间层**，用于降低 CRUSH 算法的计算复杂度和 OSD 的负载：

- **Object → PG 映射**：一个对象通过哈希算法映射到一个 PG
- **PG → OSD 映射**：一个 PG 通过 CRUSH 算法映射到一组 OSD（主 OSD + 副本 OSD）
- **作用**：如果每个对象都通过 CRUSH 定位到 OSD，计算量巨大；通过 PG 间接映射，只需在 PG 数量级做映射

```
对象 A  →  PG 10  →  OSD 0, OSD 5, OSD 12（主、副、副）
对象 B  →  PG 10  →  OSD 0, OSD 5, OSD 12
对象 C  →  PG 11  →  OSD 3, OSD 8, OSD 15
```

### 4.5 PG 的状态

每个 PG 有多个生命周期状态：

| 状态 | 说明 |
|------|------|
| **clean** | 所有副本一致且正常 |
| **active** | 处于正常服务状态 |
| **recovering** | 正在从故障 OSD 恢复数据 |
| **backfilling** | 正在回填数据到新 OSD |
| **degraded** | 副本不完整（部分 OSD 故障） |
| **inconsistent** | 数据不一致，需要修复 |
| **peering** | 正在计算哪些 OSD 持有该 PG 的副本 |

### 4.6 OSD 写入流程（完整路径）

```
1. Client 从 MON 获取最新的 Cluster Map（包含 OSD Map、PG Map、CRUSH Map）
2. Client 根据对象 ID 计算该对象属于哪个 PG
3. Client 通过 CRUSH 算法查找该 PG 的主 OSD（Primary OSD）和副本 OSD 列表
4. Client 将写入请求发送给主 OSD
5. 主 OSD 将数据写入本地存储（WAL → DB → DATA）
6. 主 OSD 将写入请求转发给副本 OSD
7. 副本 OSD 各自执行写入操作
8. 所有副本 OSD 确认写入完成后，向主 OSD 发送 ACK
9. 主 OSD 收到全部 ACK，向 Client 返回写入成功
```

### 4.7 OSD 故障恢复流程

```
1. OSD 故障，心跳停止
2. 其他 OSD 和 MON 检测到该 OSD 离线
3. MON 更新 OSD Map，标记该 OSD 为 down
4. 主 OSD 发现副本 OSD 缺失，进入 recovering 状态
5. 客户端将读取请求路由到其他存活副本
6. 从存活副本读取数据，触发 OSD 间数据重建（recovery）
7. 重建完成后，OSD 重新上线，触发数据再平衡（rebalance）
8. PG 状态恢复为 clean
```

### 4.8 OSD 常用命令

```bash
# 查看所有 OSD 状态
ceph osd stat
ceph osd tree

# 列出所有 OSD
ceph osd ls

# 查看 OSD 使用率
ceph osd df

# 查看 OSD 详细状态
ceph daemon osd.{id} status

# 查看 PG 状态
ceph pg stat
ceph pg dump
```

---

## 五、MDS（Metadata Server，元数据服务器）

### 5.1 职责

MDS 是 Ceph 文件系统（CephFS）的元数据管理组件。

**核心职责**：

- **管理目录树结构**：维护文件系统的目录层级和路径映射
- **文件元数据缓存**：缓存文件名、权限、大小、时间戳、扩展属性等
- **元数据查询加速**：快速响应目录遍历、路径解析等查询请求
- **确保元数据一致性**：在多个 MDS 之间协调元数据操作的顺序和原子性

### 5.2 使用场景

| 存储类型 | 是否需要 MDS | 说明 |
|---------|:---:|------|
| **CephFS（文件存储）** | 是 | 需要 MDS 管理目录树结构 |
| **RBD（块存储）** | 否 | 直接操作对象，无需元数据服务 |
| **RGW（对象存储）** | 否 | 通过 RADOS 直接访问，无需 MDS |
| **iSCSI（SAN 接口）** | 否 | 基于 RBD 实现，无需 MDS |

### 5.3 元数据的位置

**关键概念**：MDS 管理的元数据**不存储在 MDS 本地**，而是存储在 RADOS（OSD）中。MDS 只负责缓存和加速元数据访问。

```
MDS（内存缓存 + 计算）
  ↓ 读写
RADOS（OSD 磁盘存储）← 元数据持久化在此
```

这种设计意味着：
- MDS 本身无状态，故障后可以从 RADOS 恢复
- 元数据冗余度与 RADOS Pool 的副本数一致
- MDS 可以水平扩展（多 Active 模式）

### 5.4 MDS 部署模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **Active-Standby** | 1 个 Active + N 个 Standby | 小规模部署，简单可靠 |
| **Multi-Active** | N 个 Active 同时工作 | 大规模文件系统，不同目录由不同 MDS 管理 |

### 5.5 MDS 常用命令

```bash
# 查看 MDS 状态
ceph mds stat
ceph mds dump

# 列出 MDS 地图
ceph mds ls

# 显示 MDS 层次结构
ceph mds hierarchy
```

---

## 六、RGW（RADOS Gateway，RADOS 网关）

### 6.1 职责

RGW 是 Ceph 的对象存储网关，提供兼容 S3 和 Swift 的 HTTP RESTful API 接口。

**核心职责**：

- **HTTP 请求转换**：将 S3/Swift HTTP 请求转换为 RADOS 对象操作
- **API 兼容**：提供与 Amazon S3 和 OpenStack Swift 完全兼容的 RESTful API
- **用户认证**：管理用户身份认证、访问密钥（Access Key / Secret Key）
- **访问控制**：支持 ACL（访问控制列表）、Bucket Policy、多租户隔离
- **多站点同步**：支持跨区域对象数据同步（Multi-Site Replication）

### 6.2 数据存储路径

RGW 将对象数据直接存储在 RADOS Pool 中，无需额外的元数据服务：

```
Client（S3 API / Swift API）
  ↓ HTTP/HTTPS
RGW（网关进程）
  ↓ librados 库
RADOS Pool（OSD 存储对象数据）
```

### 6.3 RGW 部署方式

```
                  ┌─────────┐
    Client ──────→│  HAProxy│ ──────→ RGW Instance 1
                  │ Load    ──→ RGW Instance 2
                  │ Balancer─→ RGW Instance 3
                  └─────────┘
                        ↓
                   RADOS (OSD)
```

- **负载均衡**：RGW 本身无状态，通过 HAProxy/Nginx 等负载均衡器分发请求
- **多实例部署**：部署多个 RGW 实例实现水平扩展和高可用
- **多 Realm/Sync Zone**：支持跨数据中心多站点部署

### 6.4 RGW 的多租户与用户管理

RGW 管理两层用户体系：

| 用户类型 | 说明 |
|---------|------|
| **Swift 用户** | 用于 OpenStack Swift 兼容接口，每个用户有一个 Secret Key |
| **S3 用户** | 用于 Amazon S3 兼容接口，每个用户有一个 Access Key + Secret Key |
| **子用户（Sub-user）** | 由主用户创建，继承主用户的配额和权限 |

### 6.5 RGW 常用命令

```bash
# 查看 RGW 状态
ceph status
rados lspools  # 查看 RGW 创建的 pool

# 管理 RGW 用户
radosgw-admin user create --uid=testuser --display-name="Test User"
radosgw-admin user info --uid=testuser

# 管理 RGW bucket
radosgw-admin bucket list
radosgw-admin bucket stats --bucket=my-bucket
```

---

## 七、RADOS（Reliable Autonomic Distributed Object Store）

### 7.1 概念

RADOS 是 Ceph 的核心存储层，所有上层服务（RBD、RGW、CephFS）都构建在 RADOS 之上。

**核心特性**：

- **Reliable（可靠）**：通过多副本/纠删码保证数据可靠性
- **Autonomic（自治）**：自动故障检测、数据恢复、负载再平衡
- **Distributed（分布式）**：无中心调度，通过 CRUSH 算法分布数据

### 7.2 Pool（存储池）

Pool 是 RADOS 中的逻辑存储单元，类似数据库中的 Table 或命名空间：

- **每个 Pool 有独立的副本数/EC 策略**：不同 Pool 可以配置不同的可靠性级别
- **Pool → PG 映射**：每个 Pool 独立维护自己的 PG
- **Pool → OSD 映射**：通过 CRUSH Map 将 PG 映射到 OSD

```bash
# 创建 Pool
ceph osd pool create mypool 128 128  # 128 PG + 128 PG 备份

# 查看 Pool 列表
ceph osd pool ls

# 查看 Pool 详情
ceph osd pool ls detail
```

### 7.3 CRUSH（Controlled Replication Under Scalable Hashing）

CRUSH 是 Ceph 的核心算法，用于确定数据对象在集群中的存储位置，无需中心化调度。

#### CRUSH 的核心要素

| 要素 | 说明 |
|------|------|
| **Bucket（存储桶）** | 逻辑容器，用于组织 OSD 的层次结构 |
| **Host** | 主机级别故障域 |
| **Rack** | 机架级别故障域 |
| **Row** | 行级别故障域 |
| **Datacenter** | 数据中心级别故障域 |
| **Root** | 根节点，代表整个集群 |
| **Weight（权重）** | 每个 OSD 和 Bucket 的权重，影响数据分配比例 |

#### CRUSH 映射层次结构

```
root default
├── rack rack1
│   ├── host server1
│   │   ├── osd.0 (weight 1.0)
│   │   └── osd.1 (weight 1.0)
│   └── host server2
│       ├── osd.2 (weight 1.0)
│       └── osd.3 (weight 1.0)
└── rack rack2
    ├── host server3
    │   ├── osd.4 (weight 1.0)
    │   └── osd.5 (weight 1.0)
    └── host server4
        ├── osd.6 (weight 1.0)
        └── osd.7 (weight 1.0)
```

#### CRUSH 工作流程

```
1. 客户端计算对象 ID 的哈希值
2. 从 CRUSH Map 的根节点开始
3. 根据哈希值从当前节点的所有子 Bucket 中选择一个
4. 递归向下，选择权重最高的可用子 Bucket
5. 到达叶子节点（OSD）
6. 根据副本数，重复步骤选择多个 OSD 存放副本
7. 确保副本分布在不同的故障域中
```

#### CRUSH 的优势

- **无中心化**：每个客户端和服务端都可以独立计算数据位置
- **可扩展**：添加/删除 OSD 时，只有受影响的数据需要迁移
- **故障容忍**：副本分散在不同故障域，单故障域失效不影响数据可用
- **数据均衡**：CRUSH Map 权重设计确保数据均匀分布

### 7.4 数据可靠性机制

Ceph 通过两种机制保证数据可靠性：

| 机制 | 说明 | 适用场景 |
|------|------|---------|
| **多副本（Replication）** | 默认将数据复制 N 份（N 可配置） | 通用场景，读性能好 |
| **纠删码（EC）** | 将数据分成 k 块 + m 块校验码，允许丢失 k 块 | 大容量存储，节省空间 |

---

## 八、客户端接口层

### 8.1 librados

librados 是 RADOS 的客户端库，为应用程序提供直接访问 RADOS 的接口。

- **编程语言**：C/C++ 原生 API，支持 Python、Java、Ruby、PHP 等绑定
- **功能**：直接读写 RADOS 对象，构建自定义应用
- **上层依赖**：RBD、RGW、CephFS 均基于 librados 构建

### 8.2 RBD（RADOS Block Device，块设备）

RBD 是 Ceph 提供的块存储服务，将 RADOS 对象存储呈现为块设备。

**核心特性**：

- **快照（Snapshot）**：支持块设备级别快照，可创建和回滚
- **克隆（Clone）**：基于快照创建只读/可写克隆
- **Thin Provisioning（精简配置）**：按需分配空间，不预先占用
- **Cache**：支持 Writeback/Writethrough 缓存模式
- **并行 I/O**：数据条带化到多个 PG，支持高并发读写
- **虚拟化支持**：与 KVM/QEMU 深度集成，作为虚拟机磁盘

**部署方式**：

```bash
# 创建 RBD 镜像
rbd create --size 100G mypool/vm-disk

# 挂载到内核
rbd map mypool/vm-disk --name client.admin
# 设备变为 /dev/rbd0

# 格式化和挂载
mkfs.xfs /dev/rbd0
mount /dev/rbd0 /mnt/vm
```

### 8.3 CephFS（Ceph File System，文件系统）

CephFS 是 Ceph 提供的分布式文件系统，通过 POSIX 接口访问。

**核心特性**：

- **POSIX 兼容**：支持标准 POSIX 文件系统操作
- **多 Active MDS**：支持多个 MDS 同时工作（需配置 MDS 层次结构）
- **快照**：支持文件系统级别快照
- **配额（Quota）**：可对目录设置容量配额
- **挂载方式**：
  - **Kernel 挂载**：通过 Linux 内核模块挂载
  - **FUSE 挂载**：通过用户空间 FUSE 挂载（无需内核模块）

```bash
# Kernel 挂载
mount -t ceph mon1,mon2,mon3:/ /mnt/ceph -o name=admin,secret=AQAB...

# FUSE 挂载
mount.ceph mon1,mon2,mon3:/ /mnt/ceph -o name=admin,secret=AQAB...
```

### 8.4 iSCSI Gateway

iSCSI Gateway 将 Ceph RBD 镜像通过 iSCSI 协议暴露给传统 SAN 存储使用。

**核心特性**：

- **兼容性**：任何支持 iSCSI 的客户端都可以访问
- **基于 RBD**：底层使用 RBD 镜像，利用 Ceph 的副本/恢复能力
- **动态扩展**：iSCSI LUN 对应 RBD 镜像，可随时扩展

---

## 九、完整架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        客户端接口层                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐            │
│  │  CephFS  │  │   RBD    │  │   RGW    │  │ iSCSI    │            │
│  │ (MDS)    │  │(Block)   │  │(Object)  │  │ Gateway  │            │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘            │
│       └──────────────┴─────────────┴─────────────┘                  │
│                            │ librados                              │
├─────────────────────────────────────────────────────────────────────┤
│                        RADOS 层                                     │
│  ┌──────────┐  ┌──────────┐  ┌─────────────────────────────┐       │
│  │  MON     │  │  MGR     │  │        OSD Cluster           │       │
│  │ (奇数)   │  │(Active-  │  │  ┌────┐ ┌────┐ ┌────┐       │       │
│  │ Paxos/   │  │ Standby) │  │  │OSD │ │OSD │ │OSD │ ...    │       │
│  │ Raft     │  │ Dashboard│  │  │ 0  │ │ 1  │ │ 2  │        │       │
│  │ Consensus│  │ + Plugins│  │  └────┘ └────┘ └────┘        │       │
│  │          │  │ Prometheus│  │                               │       │
│  └──────────┘  └──────────┘  │  WAL + DB + DATA              │       │
│                              └─────────────────────────────┘       │
│                            │                                       │
│                    ┌───────┴───────┐                               │
│                    │   CRUSH Map   │                               │
│                    └───────────────┘                               │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 十、关键概念速查表

| 术语 | 英文全称 | 说明 |
|------|---------|------|
| **MON** | Monitor | 集群状态维护，Paxos/Raft 共识 |
| **MGR** | Manager | 运维管理，Dashboard + 插件 |
| **OSD** | Object Storage Daemon | 数据存储、复制、恢复、再平衡 |
| **MDS** | Metadata Server | CephFS 元数据服务器 |
| **RGW** | RADOS Gateway | S3/Swift 对象存储网关 |
| **RADOS** | Reliable Autonomic Distributed Object Store | 核心存储层 |
| **RBD** | RADOS Block Device | 块设备接口 |
| **CephFS** | Ceph File System | 分布式文件系统 |
| **PG** | Placement Group | 归置组，数据管理中间层 |
| **CRUSH** | Controlled Replication Under Scalable Hashing | 数据分布算法 |
| **Pool** | 存储池 | 逻辑存储单元，独立副本策略 |
| **WAL** | Write-Ahead Log | 写前日志，提升写性能 |
| **EC** | Erasure Coding | 纠删码，节省空间 |

---

## 十一、完整数据流

### 写入数据全流程

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │ 1. 获取 Cluster Map（MON）
       ▼
┌─────────────┐
│     MON     │◄── 提供 OSD Map、PG Map、CRUSH Map
└──────┬──────┘
       │ 2. 计算 PG → 定位 OSD
       ▼
┌─────────────┐
│   Client    │
│ （本地计算） │
└──────┬──────┘
       │ 3. 发送写请求到 Primary OSD
       ▼
┌─────────────────────────────────────┐
│  Primary OSD（协调者）               │
│  ┌─────────┐    ┌─────────────────┐ │
│  │  WAL    │    │   DB (RocksDB)  │ │
│  │ (NVMe)  │    │  (NVMe/SSD)     │ │
│  └────┬────┘    └────────┬────────┘ │
│       │                  │          │
│  ┌────▼──────────────────▼────────┐ │
│  │          DATA (HDD/SSD)        │ │
│  └────────────────────────────────┘ │
│                                      │
│  ┌─── 4. 同步副本 OSD ────────────┐  │
│  │   OSD 1      │   OSD 2         │  │
│  └────────────────────────────────┘ │
└─────────────────────────────────────┘
       │ 5. 全部确认后返回成功
       ▼
┌─────────────┐
│   Client    │
└─────────────┘
```

### 读取数据全流程

```
1. Client 本地计算对象 → PG → OSD（持有 Cluster Map）
2. 如果存在 Primary OSD：直接读取 Primary
3. 如果 Primary 故障：从存活副本读取
4. 返回数据给 Client
```

---

## 十二、生产部署建议

### MON 部署

- **数量**：3 个（小型）/ 5 个（中型）/ 7 个（大型）
- **磁盘**：SSD/NVMe（MON 写入频繁）
- **网络**：独立管理网络，低延迟
- **节点隔离**：MON 部署在独立物理机，不与 OSD 混部

### MGR 部署

- **数量**：3 个（至少一个 Active）
- **功能**：启用 Prometheus + Dashboard + Crash 模块

### OSD 部署

- **日志分离**：WAL 和 DB 使用 NVMe SSD，DATA 使用 HDD 或 SSD
- **SSD:HDD 比例**：1:3~5
- **网络**：数据网络高带宽（10/25/100GbE），管理网络独立
- **磁盘类型**：企业级 SAS/SATA SSD，避免消费级磁盘

### MDS 部署（CephFS）

- **模式**：小型部署用 Active-Standby，大型部署用 Multi-Active
- **多 Active**：配置 MDS 层次结构，不同目录由不同 MDS 管理

### RGW 部署

- **负载均衡**：RGW 无状态，通过 HAProxy/Nginx 负载均衡
- **多实例**：至少 2 个实例，配合健康检查
- **多站点**：跨区域部署需要配置 Multi-Site Replication

---

*文档版本：v1.0*  
*最后更新：2026-08-07*