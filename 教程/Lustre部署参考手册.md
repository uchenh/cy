# Lustre部署参考手册


<!-- 第 1 页 -->

1
Lustre 部署与使用参考手册


<!-- 第 2 页 -->

2
目录
第一篇 基础与选型 ..........................................................3
第 1 章 Lustre 概述 ................................................... 3
第 2 章 系统架构与核心概念 ............................................. 5
第 3 章 Lustre 与 Ceph 选型对比 ........................................9
第二篇 部署规划 .......................................................... 11
第 4 章 版本与兼容性 .................................................. 11
第 5 章 环境与拓扑规划 ................................................ 14
第三篇 部署实操 .......................................................... 18
第 6 章 RHEL 系（Rocky 8.10）虚拟机部署 ............................... 18
第 7 章 Ubuntu 24.04 部署方法论 .......................................23
第 8 章 Ubuntu 3 节点集群实测部署记录 ..................................29
第 9 章 单机 All-in-One 部署 ......................................... 38
第 10 章 扩容：新增 OSS 节点与数据盘 ................................... 45
第四篇 运维与排障 .........................................................49
第 11 章 性能调优 .....................................................49
第 12 章 日常运维 .....................................................51
第 13 章 故障排查速查总表 ............................................. 53
附录 .................................................................... 57
附录 A 部署前检查清单 ................................................. 57
附录 B 实测经验教训汇总 ............................................... 58
附录 C 术语与常用命令速查 ............................................. 59
附录 D 参考资料 .......................................................61


<!-- 第 3 页 -->

3
第一篇 基础与选型
第 1 章 Lustre 概述
1.1 什么是 Lustre
Lustre 是一个开源的高性能并行分布式文件系统（Parallel Distributed File
System）：它把一份文件拆成多块、同时写入多台服务器的多个磁盘，让成千上万个计算节
点像访问本地目录一样并发读写同一份共享数据，聚合带宽随存储节点数量线性增长。名字取
自 Linux + Cluster，最初由 Cluster File Systems 公司开发。
它要解决的问题：普通文件服务器（NFS）撑不住几千个节点同时读一个大文件——单台机器的
磁盘带宽就是上限。Lustre 的思路是"让 N 台机器一起干活，带宽乘以 N"。其核心目标：
 高聚合吞吐：通过数据条带化分布到多个对象存储目标（OST），聚合带宽随存储节点线
性扩展；
 海量可扩展容量：单文件系统可支持数十 PB 乃至 EB 级容量；
 POSIX 兼容的全局统一命名空间：所有客户端看到一致的目录树；
 高可用（HA）：关键组件支持故障切换与自动恢复。
1.2 起源与历史
阶段 时间 关键事件
起源 1999–2003 Peter Braam 创立 Cluster File Systems 公司，Lustre 在
Sandia/LLNL 等超算中心落地
Sun 时期 2007 Sun Microsystems 收购 Cluster File Systems，Lustre 进入商
业化推广
Oracle 时期 2010 Oracle 收购 Sun，Lustre 发展一度放缓
Whamcloud 时期 2010 原核心团队创立 Whamcloud，专注 Lustre 研发与社区
DDN 时期 2019 高性能存储厂商 DDN（DataDirect Networks）收购 Whamcloud，持
续投入主线开发
社区化 持续 OpenSFS 协调社区路线图，源码托管于 Whamcloud git
1.3 典型应用场景与适用边界
适合（多机并发读大文件、需要高聚合带宽）：
 超级计算 / HPC：Top500 中大量超算系统采用 Lustre 作为共享并行存储；
 AI 训练与大数据：大规模训练数据集（图像、语料、特征）的并发读取与 Checkpoint
写入；
 影视渲染 / 科学仿真：海量中间文件与结果文件的高并发写入与共享访问；
 气象、能源、基因等科研：PB 级科研数据的长期共享与高吞吐分析。
适用边界：


<!-- 第 4 页 -->

4
 部署运维复杂：内核模块、LNet、版本与内核强绑定；
 小文件 / 元数据密集场景不占优（锁争用 + MDT 瓶颈，需 DNE 缓解）；
 无原生纠删码，冗余靠后端 RAID/ZFS 或 FLR 镜像，容量开销大；
 不适合跨地域（低延迟局域网设计）；
 加 OST 在线扩容容易，移除 OST 必须先迁移数据。
1.4 主要版本演进
 Lustre 1.x：早期原生版本，已退出历史舞台。
 Lustre 2.x：当前主线，基于客户端/服务端分离的现代化架构。
 2.12 / 2.14：经典稳定版本；
 2.15.x（LTS）：当前稳定 LTS 分支（如 2.15.8），RHEL 系首选；
 2.16.x：引入 IPv6 大 NID（LU-10391）、目录遍历优化、RHEL 9.4 Server 支
持，首次官方支持 Ubuntu 24.04；
 2.17.0：新增 Hybrid IO（LU-18033）、动态 LNet NID 配置（LU-18815）、
Nodemap 多租户增强（LU-17431）；
 2.18.0：开发中。
实践建议：生产环境优先选择 LTS 版本（如 2.15.8）以获得长期稳定支持；Ubuntu
24.04 等 6.x 内核平台需 2.16+。
1.5 与其他并行/分布式文件系统概览
系统 类型 定位 与 Lustre 对比要点
GPFS / Spectrum
Scale（IBM）
闭源并行文件系统 企业级
HPC/AI
功能全面、商业化支持强；Lustre 开
源、超算生态更成熟、聚合带宽上限更高
BeeGFS
（ThinkParQ）
开源并行文件系统 HPC/AI 部署轻量、易上手；Lustre 在超大规模
（PB~EB）、生态工具上更成熟
CephFS 开源分布式文件系统 云/通用存储 统一存储（对象+块+文件）；Lustre 在
纯 HPC 高带宽场景性能更优（详见第 3
章）
GlusterFS 开源分布式文件系统 通用/文件同
步
无中心化 MDS，扩展简单但 HPC 高并发
元数据性能弱于 Lustre
小结：Lustre 的核心优势是极致聚合带宽、超大规模可扩展、成熟的 HPC 生态；劣势是部
署与运维复杂度较高、对内核版本有较强绑定。


<!-- 第 5 页 -->

5
第 2 章 系统架构与核心概念
2.1 分离式架构总览
Lustre 采用分离式架构，将元数据服务、对象数据存储、管理配置、客户端访问彻底解耦——
"文件叫什么、在哪"（元数据）与"文件内容存哪"（数据）是两条完全独立的链路：
+-----------------------------+
| Clients |
| (lustre.ko 内核模块 / llapi)|
+--------------+--------------+
| LNet (TCP/o2ib/RoCE)
+----------------------+----------------------+
| | |
+------v------+ +-------v-------+ +-------v-------+
| MGS | | MDS | | OSS |
| (MGT 配置) | | (MDT 元数据) | | (OST 数据对象) |
+-------------+ +-------+-------+ +-------+-------+
| |
inode/布局信息 条带化对象数据
图书馆类比（帮助快速建立直觉）：
组件 全称 角色 图书馆类比
MGS Management Server 集群"大脑"，存配置日志，挂载入口 图书馆总台
MGT Management Target MGS 的存储目标（几 GB 够，必须高可用） 总台登记簿
MDS Metadata Server 管目录树、inode、权限、文件布局 目录卡片管理员
MDT Metadata Target MDS 的存储目标（建议 SSD/NVMe） 卡片柜
OSS Object Storage
Server
提供数据服务，每台管 1~N 个 OST 书架管理员
OST Object Storage
Target
存数据对象，后端 ldiskfs/ZFS 书架
Client 客户端 内核模块 lustre.ko，挂载为本地目录 读者
LNet Lustre Networking 网络抽象层（TCP/InfiniBand/RoCE） 馆内通道
关键机制：数据 IO 直连。客户端读写文件内容不经过 MDS，而是按 MDS 给出的布局信息
直连 OSS。元数据走窄路、数据走宽路，互不堵塞——这是高吞吐的第一个来源。
2.2 各组件职责
MGS / MGT（管理节点）
 一个 MGS 可管理多个文件系统（多个 fsname）；MGT 上保存各文件系统的配置日志
（目标列表、NID、failover 伙伴等）；
 客户端挂载流程：先连 MGS → 拉取对应 fsname 的 config log → 据此连接 MDS/MDT
与 OSS/OST；
 MGT 通常较小（几 GB 即可），但必须高可用——它是集群挂载的前提。
MDS / MDT（元数据服务器）


<!-- 第 6 页 -->

6
 负责所有元数据操作：路径解析、inode 分配、权限检查、扩展属性、布局信息
（striping 描述）；
 单个 MDT 可能成为元数据瓶颈，可通过 DNE（Distributed Namespace，分布式命名
空间） 将目录树分布到多个 MDT（2.4+ 支持）；
 MDT 后端：传统 ldiskfs（经补丁增强的 ext4）或 ZFS（校验和、快照、压缩）。
OSS / OST（对象存储服务器）
 每个 OSS 通常管理 1~8 个 OST；文件被切分为对象写入 OST，每对象有唯一
FID/OID；
 OST 后端为本地文件系统（ldiskfs 或 ZFS），单 OST 容量常见数 TB 到数十 TB；
 OST 之间无直接耦合，增加 OST 即可线性扩展容量与带宽。
Client（客户端）
 需加载 Lustre 内核模块（lustre.ko、lvfs.ko、osc.ko、mgc.ko 等），挂载命令
mount -t lustre <MGS NID>:/<fsname> /mnt/lustre；
 用户态工具：lfs（条带/配额/布局）、lctl（底层参数与调试）、liblustreapi（应用
级调用）。
LNet（Lustre Networking）
 屏蔽底层网络差异，支持的网络类型：
 tcp：标准 TCP/IP（socklnd，监听 988 端口），兼容性最好，实验首选；
 o2ib：InfiniBand 上的 OFA 接口，低延迟高带宽，HPC 生产首选；
 tcp over RoCE：基于融合以太网的 RDMA。
 NID = IP 地址 + 网络类型后缀（如 10.0.0.1@tcp）；一个节点可配置多个 NI；
 跨 LNet 子网互通需部署 LNet Router；配置方式：lnetctl net add（动态，推荐）或
modprobe.d 的 options lnet networks=...（静态）。
2.3 条带化（Striping）与文件布局
条带化是 Lustre 性能的核心：把一个文件拆成多块同时写多个磁盘。例如 16 MiB 数据拆
成 4 段写 4 块盘，吞吐接近单盘 × 4。核心参数：
参数 含义 典型取值
stripe_count 文件使用的 OST 数量（条带宽度） 默认 1；大文件常设 -1（用满所有
OST）
stripe_size 每个条带单元大小（轮转写入 OST 的单位） 默认 1 MiB；大顺序 IO 可增大至
4/8 MiB
stripe_offset 起始 OST 索引 -1 表示由 MDS 自动选择（轮转均
衡）


<!-- 第 7 页 -->

7
布局语义：文件前 stripe_size 字节写入起始 OST，接下来写入下一个 OST，依次轮转；写
满 stripe_count 个 OST 后回到第一个。单个大文件的吞吐即可聚合多个 OST/OSS 的带
宽。
查看与设置（lfs 命令）：
```bash
# 查看文件/目录布局
lfs getstripe /mnt/lustre/bigfile.dat
lfs getstripe -d /mnt/lustre/dataset/ # 目录默认布局
# 创建文件时指定条带
lfs setstripe -c 4 -S 4M -i -1 /mnt/lustre/bigfile.dat
# -c 4 : 使用 4 个 OST
# -S 4M: 条带单元 4 MiB
# -i -1: 起始 OST 由 MDS 自动选择
# 设置目录默认布局（该目录下新建文件继承）
lfs setstripe -c -1 -S 1M /mnt/lustre/dataset
# -c -1 : 使用所有可用 OST（最大并发）
PFL（Progressive File Layout，渐进式文件布局）：文件在生命周期内访问模式会变化
（创建时小、后期变大），固定布局不经济。PFL 将文件划分为多个组件，每个组件采用不
同条带参数：
lfs setstripe -E 1M -c 1 -E 1G -c 4 -E -1 -c -1 /mnt/lustre/pfl_file
# 前 1M 用 1 个 OST；1M~1G 用 4 个 OST；>1G 用全部 OST
FLR（File Level Redundancy）：通过 lfs mirror 创建镜像/纠删码副本，提升数据可靠
性（类似 RAID 跨 OST）。
2.4 锁机制：LDLM
Lustre 通过 LDLM（Lustre Distributed Lock Manager，分布式锁管理器） 保证多客
户端数据一致。类比多人协作编辑文档：客户端只对文件某个字节区间持锁（Extent 锁），
改不同段落可并行；改同一段要等对方"写完保存"——持有者收到回调（callback）后刷回脏
数据并释放锁。
锁模式从松到严：
模式 含义 兼容性
EX（Exclusive） 独占写 不与其他任何模式共存
PW（Protected Write） 保护写 与 PR/NL 兼容，排斥 EX/PW
PR（Protected Read） 保护读 与 PW/PR/NL 兼容，排斥 EX
NL（Null，占位锁） 不执行实际锁保护 与所有模式兼容
```


<!-- 第 8 页 -->

8
 Extent 锁：针对文件字节区间的锁。多个客户端读写不同区间可并行——这是高聚合带宽
场景下仍能保持一致性的关键；
 Inode 锁：保护文件布局、权限、size 等元数据，变化时失效其他客户端缓存；
 授予 / 排队 / 回收：无冲突直接授予；冲突则排队等待；另一客户端需要冲突锁时，原
持有者收到 cancel callback，刷回脏数据并释放锁；
 一致性语义：Lustre 默认提供 POSIX 一致性（close-to-open）——文件关闭后重新打
开能看到其他客户端的最新写入；打开期间的可见性需显式 flock/fcntl 加锁保证；
 性能影响：锁争用（如大量小文件并发写同一目录）会成为瓶颈，部署时应分散热点目
录。
2.5 故障恢复（Failover / Recovery）
 active/passive 双机共享存储：MDS/OSS 故障由备用节点挂载同一后端盘秒级接管
（生产用 Pacemaker + Corosync 编排，mkfs.lustre --servicenode=<主> --
servicenode=<备> 声明故障切换伙伴）；
 客户端重放：服务端故障期间客户端 IO 阻塞并进入 recovery 状态、持续重连；服务
恢复后重放（replay）未完成操作日志，事务不丢；
 超过恢复窗口（默认最长 300s）服务端会驱逐（evict）客户端，重新挂载即可恢复；
 脑裂：靠共享存储 fencing 机制避免双写；
 常见故障场景：MDT 盘损坏（依赖共享存储 failover 或 ZFS 快照恢复）、OST 离线
（其他条带仍可用）、客户端 evicted（检查网络/LNet 后重挂载）。


<!-- 第 9 页 -->

9
第 3 章 Lustre 与 Ceph 选型对比
两者定位不同：Lustre 是面向 HPC/超算的并行文件系统，Ceph 是面向云与通用的统一存
储（块/对象/文件三合一）。理解这一点是选型的前提。
3.1 八维度总览
维度 Lustre Ceph
架构定位 分离式并行文件系统（元数据/数据物理隔离） 统一去中心化存储（RADOS 底座）
元数据 MDS/MDT 专职；单 MDT 或 DNE 分布式 MDS 集群（CephFS）；多 active
动态子树分片
数据分布 文件对象条带化到 OST（布局由 MDT 记录） CRUSH 算法伪随机分布到 OSD（多副
本/EC）
最擅长 极致聚合带宽、大规模顺序 IO、HPC 统一存储、云原生、内建冗余自愈
扩展性 加 OSS/OST 近线性扩带宽；加 MDT 扩元数据 加 OSD 自动均衡；加 MDS 扩元数据
高可用 组件级 HA（Pacemaker）+ 后端冗余；无内建自
愈
多副本/EC + 自动自愈；无单点
一致性 POSIX + LDLM 区间锁；close-to-open RADOS 强一致 + CephFS caps 协调
运维 定制内核、组件多、学习曲线陡 Cephadm/Rook 自动化 +
Dashboard
3.2 架构与数据路径差异
 Lustre 的数据路径更"短"：MDS 只在 open/close/stat 等元数据操作时介入，实际
读写由客户端与 OST 直连，避免元数据服务器成为 I/O 瓶颈——非常适合大文件、高带
宽场景；
 Ceph 的寻址是"计算出来的"：CRUSH 把对象名、副本数、故障域规则作为输入，确定性
地算出存放位置，扩容时数据按规则增量再平衡，无需全局查表；
 扩展单元不同：Lustre 以 OST/OSS 带宽为单位，Ceph 以 OSD/PG 为单位；两者都可
横向扩展至数十 PB/EB 级。
3.3 适用场景与选型建议
用例 推荐 说明
百亿亿次超算并行文件系统 Lustre 单文件系统数十 PB、数百 OSS，
顺序 IO 带宽优先
AI 训练数据湖（大文件读） Lustre（或 Ceph 配高性能 OSD） 大数据集顺序读、检查点写入
OpenStack/K8s 云存储后端 Ceph RBD + RGW + CephFS 一体化
企业对象存储（S3 兼容） Ceph（RGW） 多租户、EC 省空间
中小团队通用 NAS/共享 Ceph（CephFS） 统一、易扩、自愈
需要强 POSIX 语义 + 极致
带宽
Lustre 客户端缓存 + LDLM 锁优化并发
选型结论：
1. 选 Lustre，当：核心诉求是"超高聚合带宽 + 大文件并行顺序 IO + POSIX 强一致 +
HPC 生态（Slurm/MPI/并行 HDF5）"，且团队具备 HPC 存储运维能力、愿为 HA 投入
共享存储与 Pacemaker；


<!-- 第 10 页 -->

10
2. 选 Ceph，当：核心诉求是"块/对象/文件统一 + 云原生 + 内建冗余自愈 + 平滑扩容
+ 通用混合负载"，希望自动化部署与可视运维；
3. 两者并存（推荐组合）：大型科研/云环境常见"Lustre 做 HPC 高性能并行文件系统
层、Ceph 做通用统一存储/对象归档层"，兼顾极致性能与统一管控。
3.4 高可用、一致性与运维差异要点
 冗余与自愈：Lustre 无内建跨 OST 副本，数据可靠性依赖后端 RAID/ZFS——这是有意
的设计取舍：将冗余责任下放到存储层，避免副本写放大，换取更高聚合带宽；Ceph 内
建多副本/EC 与自动 backfill 恢复，OSD 故障自动重建；
 一致性：Lustre 是"POSIX + 区间锁"的硬保证，天然契合 HPC 分块聚合写；CephFS
通过 caps 在"客户端缓存性能"与"一致性窗口"之间留出可调空间；
 部署自动化：Lustre 多为手动 + 脚本（mkfs.lustre 逐节点配置），服务端依赖补丁
内核；Ceph 有 Cephadm/Rook 声明式自动化与 Dashboard；
 生态与人才：Lustre 深耕 HPC/超算（与 Slurm/MPI 集成成熟，AWS FSx for
Lustre 托管）；Ceph 社区庞大、文档培训丰富、云原生黏性强。
3.5 混合部署数据同步
若同时采用 Lustre 与 Ceph，推荐的数据迁移/同步工具：
 rsync：通用数据同步，适合中小规模批量迁移；
 Robinhood 策略引擎：Lustre 大规模数据分层与归档自动化；
 FUSE 网关：实现 Ceph S3 对象与 POSIX 文件系统的桥接访问。


<!-- 第 11 页 -->

11
第二篇 部署规划
第 4 章 版本与兼容性
4.1 版本匹配三要素
Lustre 的服务端代码与 Linux 内核深度耦合，服务端必须使用 Whamcloud 打过补丁的定
制内核（或自行编译内核模块）。因此「系统发行版 + Lustre 版本 + 内核版本」三者必
须绑定，且：
 服务端三个角色（MGS/MDS/OSS）必须使用完全相同的 Lustre 版本与内核版本，否则
lctl dl 会出现版本不匹配、目标挂载失败；
 内核与内核模块严格一一对应：uname -r 必须等于 kernel-devel/kernel-headers
版本，也必须等于 kmod/lustre 模块编译所针对的内核版本。系统更新内核后 Lustre
模块将无法加载——安装后要锁定内核；
 客户端与服务端建议同版本（官方允许客户端不高于服务端一个版本线内，实验环境请保
持完全一致）。
4.2 RHEL 系推荐组合（CentOS/Rocky/Alma/RHEL）
方案 发行版 Lustre 版本 服务端内核（示例） 结论
A（推荐） Rocky/Alma/RHEL
8.10
2.15.8（当前
LTS）
4.18.0-
553.82.1.el8_lustre
活跃维护、文档
多，首选
B（旧环境） CentOS 7.9 2.12.9（前代
LTS）
3.10.0-
1160.49.1.el7_lustre
大量旧教程基于
此；EOL 后仅学
习用
C（新内核） RHEL/Rocky 9.x 2.16.x /
2.17.x
5.14.0-
<build>.el9_lustre
服务端上 EL9 需
2.16+，社区仍优
先推荐 LTS
客户端变体 客户端可放宽 2.15.x 发行版原装内核
（patchless）
2.15 客户端支持
EL8/EL9、
SLES15 SP6、
Ubuntu 22.04
Whamcloud 预编译仓库路径规律：
https://downloads.whamcloud.com/public/lustre/lustre-<版本>/<发行版标
识>/{server|client|patchless-ldiskfs-server}/
https://downloads.whamcloud.com/public/e2fsprogs/{latest|<版本号>}/<el 版本>/
例如方案 A：https://downloads.whamcloud.com/public/lustre/lustre-
2.15.8/el8.10/server/ 与 .../client/；e2fsprogs 用
https://downloads.whamcloud.com/public/e2fsprogs/latest/el8/。


<!-- 第 12 页 -->

12
服务端常用包：lustre（元包）、kmod-lustre、kmod-lustre-osd-ldiskfs、lustre-osd-
ldiskfs-mount、lustre-resource-agents（HA 用）；客户端：lustre-client。必须同时安
装 Whamcloud 定制的 e2fsprogs（版本号带 wc 后缀），否则 mkfs.lustre 格式化
ldiskfs 会失败。
⚠ 安装前先登录 downloads.whamcloud.com 核对仓库实际内容，确认内核包名与版本
号后再执行安装，不要照抄旧文档的包名。
什么时候需要自己编译内核模块：当目标发行版/内核在 Whamcloud 仓库没有预编译服务端
包时（例如 Ubuntu 服务端、较新的 RHEL 小版本）。编译要点：
```bash
# 1) 依赖：gcc gcc-c++ make libtool autoconf kernel-devel-$(uname -r) + Whamcloud 定
制 e2fsprogs
# 2) 源码：从 downloads.whamcloud.com 的 SRPMS 子目录下载 lustre-<ver>.src.rpm
./configure --with-linux=/usr/src/kernels/$(uname -r) # 指向 kernel-devel 安装的内核
头文件
make && make modules
make modules_install # 安装到 /lib/modules/<uname -r>/
make install # 安装用户态工具（mkfs.lustre、lctl、lfs 等）
depmod -a # 重建模块依赖，否则 modprobe 找不到新模块
modprobe lustre && lsmod | grep lustre
lctl version # 输出版本号即为成功
⚠ 若 modprobe 报 Invalid module format，多为内核头文件与运行内核版本不一致，
重装 kernel-devel-$(uname -r) 后重编。源码编译首次通常要半天到一天，实验环境优
先用预编译包。
4.3 Ubuntu 24.04 支持现状与版本警告
维度 支持现状
客户端 从 Lustre 2.16 起官方支持，属 patchless 客户端——无需补丁内核
服务端 从 Lustre 2.16 起列入官方支持矩阵；2.16 ChangeLog 将内核 6.8.0-38、
6.10.0-15 列为 server primary kernels
预编译包 官方不提供 Ubuntu 预编译服务端 .deb，服务端必须源码编译（见第 7.3 节）
官方 apt 仓
库
无官方通用 apt 仓库；客户端可从 Azure Managed Lustre / Google Cloud
Artifact Registry / 第三方镜像获取（见第 7.2 节）
重要警告：Ubuntu 24.04 上绝对不要使用 Lustre 2.15：
Lustre 版本 类型 Ubuntu 24.04（kernel 6.x）支持 结论
2.15.x LTS 不支持（官方明确 2.15.5 不兼容 kernel
6；客户端最高只覆盖 Ubuntu 22.04）
不要用
2.16.x feature 分支 支持（客户端 primary 6.8.0-35；服务端
primary 6.8.0-38）
推荐
2.17.x 最新 feature 分支 支持 推荐
```


<!-- 第 13 页 -->

13
互操作提示：方案 A（Rocky 服务端 2.15.8 + Ubuntu 客户端 2.16.x）属于官方支持的
互操作范围——2.16 ChangeLog 声明覆盖 "Latest 2.15.X"。
4.4 GA 内核 vs HWE 内核（Ubuntu）
Ubuntu 24.04 的镜像（尤其是 Azure 等云厂商镜像）默认安装的内核分两类，必须分清：
类型 代表包 支持周期 与 Lustre 的配合
GA / LTS 内核 linux-image-generic
（6.8.0-*-generic）
随 24.04 LTS 长期维
护
各客户端仓库的模块均
针对 6.8 线提供，推荐
HWE 内核 linux-image-azure（Azure
默认）、linux-image-
generic-hwe-24.04
每个 HWE 内核仅支持约
6 个月
Lustre 适配跟不上
HWE 迭代节奏，HWE 一
升级模块就编译失败
建议：所有节点统一切换回 GA 内核（6.8） 并锁定版本：
```bash
# 1) 查看当前内核与已安装的内核包
uname -r
dpkg -l | grep -E 'linux-(image|headers)' | awk '{print $2}'
# 2) 安装 GA 内核与配套头文件
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic
# 3) 移除默认的 HWE 内核元包（Azure 镜像为 linux-image-azure；其他平台按实际包名调整）
sudo apt remove -y linux-image-azure linux-headers-azure
# 4) 重启进入 6.8.x-generic 内核后，锁定内核
sudo apt-mark hold linux-image-generic linux-headers-generic
⚠ 实测补充（见第 8 章）：若已先装 zfs-dkms 再想切内核，安装会卡在"为新内核构
建 ZFS 模块"（ZFS 2.2.2 仅兼容内核 ≤6.6），dpkg 锁被长期占用。正确顺序是先切
6.8 并 purge HWE 内核，再装 ZFS。卡死修复：sudo pkill -f "dkms build -m zfs"
+ sudo dpkg --configure -a。
```


<!-- 第 14 页 -->

14
第 5 章 环境与拓扑规划
5.1 硬件规划原则
 组件分离：MGS、MDT、OSS 建议物理分离或至少独立磁盘，避免 IO 互相干扰；小集群
可合并 MGS+MDT 于同机；
 MGS：小容量、高可用即可（独立盘或分区，几 GB 足够）；
 MDT：元数据随机小 IO 密集，建议低延迟 SSD/NVMe；DNE 下可多个 MDT；
 OSS/OST：大容量 HDD 或 SSD；每 OST 4~16 TB 常见，单机 OST 数受内存限制；
 网络：生产首选 o2ib/RoCE；管理网与存储网分离；实验环境 @tcp 即可；
 后端文件系统：ldiskfs（高性能、成熟）或 ZFS（校验、压缩、快照，较吃内存）。
5.2 拓扑形态
单节点部署（学习/功能验证）——MGS+MDS+OSS+Client 全部合并到一台：
┌────────────────────────────── 一台 VM ──────────────────────────────┐
│ MGS+MGT(/dev/sdb) MDS+MDT(/dev/sdc) OSS+OST0(/dev/sdd) OST1(/dev/sde) │
│ ↑ 本机内部 LNet @tcp │
│ Client（本机） 挂载 <本机 IP>@tcp:/lustre → /mnt/lustre │
└────────────────────────────────────────────────────────────────────┘
多节点部署（推荐，贴近生产）——3 台：mds1（MGS+MDS）、oss1（OSS）、client1（客户
端）：
Management/Data 网络：192.168.10.0/24（LNet @tcp，端口 988）
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ mds1 │ │ oss1 │ │ client1 │
│ 192.168.10.11│ │ 192.168.10.12│ │ 192.168.10.13│
│ MGS + MGT │ │ OSS + OST0 │ │ Client │
│ MDS + MDT │ │ OSS + OST1 │ │ /mnt/lustre │
│ /dev/sdb=MGT │ │ /dev/sdb=OST0│ └──────┬───────┘
│ /dev/sdc=MDT │ │ /dev/sdc=OST1│ │
└──────┬───────┘ └──────┬───────┘ │
└──── 988/tcp ◄────────┴──────── 988/tcp ◄─────┘
加一台 oss2（OST2/OST3）即可扩展为 4 节点，命令与 oss1 完全一致（换 --index）。
单节点 vs 多节点的命令差异：
环节 多节点 单节点
MGS NID 引用 写 MDS 节点 IP：--
mgsnode=192.168.10.11@tcp
写本机 IP
mount -t
lustre
<MGS-IP>@tcp:/lustre 同样写法，指向本机 IP
LNet 配置 每台机器各自 lnetctl net add 只配一次


<!-- 第 15 页 -->

15
部署顺序 mds1 → oss1 → client1 同机按 MGS→MDT→OST→Client 顺
序
排障重点 防火墙/端口/主机名解析 磁盘设备名
5.3 VM 资源配置建议
资源 最小可跑（1 台单节点） 推荐（3 台多节点）
CPU（每 VM） 2 vCPU MDS 4 vCPU / OSS 4 vCPU /
Client 2 vCPU
内存（每 VM） 4 GB MDS 8 GB / OSS 8 GB / Client
4 GB
系统盘 20 GB（thin） 20 GB（thin）
数据盘 MGT 1 GB×1、MDT 10 GB×1、OST 20 GB×2 按角色：MDS 1+10 GB；OSS 20
GB×2~4
网卡 1 块虚拟网卡 1 块（管理+数据共用）或 2 块
（进阶）
? 角色可以合并：MGS 与 MDS 常放同一节点；资源紧张时四角色全合一（单节点形
态）。
5.4 磁盘规划
 控制器选择：KVM 用 virtio-scsi；VMware 用 PVSCSI；VirtualBox 用 SATA
（AHCI）或 NVMe；
 每类 target 一块独立虚拟盘：MGT / MDT / 每个 OST 分别对应独立块设备（如
/dev/sdb=MGT、/dev/sdc=MDT、/dev/sdd=/dev/sde=OST）。不要在系统盘上为多个
target 建 LVM 池或把盘做 RAID；
 分区/LVM：实验环境建议直接对整个块设备 mkfs.lustre（最省事）；若分区，务必每
个 target 独占一个分区并记录 UUID；
 稀疏盘：qcow2 稀疏文件省宿主机空间，但首次写满时会突然变慢，介意就选
raw/thick；
 ⚠ 绝对不要对运行中的 Lustre 节点做整机快照/回滚。Lustre 目标
（MGT/MDT/OST）与 MGS 上记录的配置日志必须严格一致，回滚会让「目标磁盘内容」
与「MGS 记录的生成代次」失配，导致整机无法挂载或数据损坏。备份请用 Lustre 原
生手段与文件级工具（如 rsync），而非 VM 快照。
5.5 网络规划
 带宽：实验 1GbE 足够；生产 HPC 存储网标配 25~100 GbE 或 InfiniBand；
 RDMA / o2ib 取舍：


<!-- 第 16 页 -->

16
网络类型 NID 示例 VM 里是否可用 说明
tcp（socklnd，默认端
口 988）
192.168.10.11@tcp ✅ 直接可用 实验首选，配置最简单
o2ib
（InfiniBand/RoCE）
192.168.10.11@o2ib ⚠ 需 SR-IOV 或 PCI
直通 HCA
虚拟化环境极少用
结论：实验环境一律用 @tcp。VM 里做 o2ib 需要直通 IB HCA，投入产出比极低。
 防火墙：LNet socklnd 监听 TCP 988 端口。实验环境直接关防火墙；若要放行，开放
988/tcp 即可（RHEL 系 firewalld / Ubuntu ufw，见第 13 章排障表）。
5.6 虚拟化平台注意事项
通用（所有平台）
 时钟漂移是最大隐患：Lustre 客户端与服务器之间的 RPC 有时间戳/恢复机制，时钟偏
差会导致客户端被 evicted。每个节点必须配置 chrony/NTP 指向同一时间源；
 关闭 SELinux 或正确打标签（RHEL 系）：实验环境建议 SELINUX=disabled（Lustre
内核模块与 SELinux 标签体系兼容性差）；Ubuntu 默认 AppArmor 一般无需处理；
 避免内存气球与 swap：内存气球回收、swap 抖动会让 Lustre 处理超时进而触发
evict：
swapoff -a && sed -i '/ swap /s/^/#/' /etc/fstab
 CPU 绑核：非必要；单核过载是时钟/超时问题的常见诱因。
KVM/QEMU：网卡、磁盘都选 virtio（virtio-net / virtio-scsi），用 virt-install
或 virt-manager 创建。
VMware（Workstation/Fusion/ESXi）：磁盘控制器选 PVSCSI，网络选 VMXNET3（不要
用 e1000）。VMXNET3 默认开启 TCP offload，个别内核/驱动组合下会导致大包丢包（表
现为随机超时），可用 ethtool -K ens192 tx off rx off 临时关闭验证；ESXi 中禁用内
存气球或给足预留内存。
VirtualBox：磁盘用默认 SATA/AHCI；虚拟网络建议选 Bridged（桥接），所有 VM 与宿
主机同网段、静态 IP（NAT 的 988 端口转发麻烦）；性能上限较低，适合学习验证，不适
合性能测试。
5.7 物理机生产档配置
按生产实际标准给配置，不是"能跑"的最低配置。若机器达不到，部署仍可完成，但性能/
并发结果仅作参考。
项 生产实际配置 生产依据
CPU 8~16 核，主频 ≥ 3.5 GHz MDS 元数据操作吃单线程延迟，OSS 数据路径吃
多核并行


<!-- 第 17 页 -->

17
内存 32~64 GB ECC ZFS ARC 默认上限 50% 内存；每 1 TB OST 容
量约需 ≥ 1 GB ARC，MDT 另需 4~8 GB 起步
系统盘 480 GB SSD（建议双盘 RAID1） 系统 + 编译产物
MGT 16 GB NVMe，独立盘 集群挂载的前提，必须低延迟、高可靠
MDT 500 GB~1 TB NVMe 元数据是随机 4K 小 IO；容量约为后端总数据量
的 1/100~1/1000
OST 2~4 块 × 2~4 TB，ZFS mirror/RAID-Z
或 HDD RAID5/6
生产单 OST 常见 4~16 TB；绝不做单盘——生产
底线是 mirror 或 RAID-Z
网络 10 GbE 起步，优先 RoCE/InfiniBand 网络带宽必须匹配后端聚合吞吐
其他 ECC 内存、双电源/UPS、chrony/NTP 时钟漂移会直接导致客户端被 evict
生产形态参考（真正的生产部署为多节点）：MGS+MDS 2 台 active/passive（16 核/64
GB，MDT 用 NVMe 共享存储或 DNE 多 MDT，Pacemaker 故障切换）；OSS 按吞吐目标 N
台（16 核/64~128 GB，每台 2~4 OST × 4~16 TB 带 RAID）；网络 RoCE/IB 25~100
GbE，管理网与存储网分离。
单机 All-in-One 无法满足生产的高可用与组件隔离要求，其意义在于用接近生产的配置
验证架构与命令流程，为生产集群铺路。


<!-- 第 18 页 -->

18
第三篇 部署实操
第 6 章 RHEL 系（Rocky 8.10）虚拟机部署
本章命令默认在 Rocky/Alma/RHEL 8.10 + Lustre 2.15.8（第 4 章方案 A） 上执
行；CentOS 7.9 + 2.12.9 把 dnf 换成 yum、仓库路径换成对应 el7 路径即可。文中
ens192 是示例网卡名，请以 ip link 为准。
6.1 环境准备（所有节点）
```bash
# 1) 主机名（在每台节点上分别执行）
hostnamectl set-hostname mds1 # oss1 / client1 各自设置
# 2) /etc/hosts 全节点一致（Lustre 通信建议全走静态 IP，不依赖 DNS）
cat >> /etc/hosts <<EOF
192.168.10.11 mds1
192.168.10.12 oss1
192.168.10.13 client1
EOF
# 3) 时间同步（关键！全节点指向同一时间源，避免客户端被 evict）
dnf install -y chrony
systemctl enable --now chronyd
chronyc sources -v # 出现 ^* 即同步成功
# 4) 关闭 SELinux 与防火墙（实验环境）
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld
# 若不想关防火墙，至少放行 LNet 端口：
# firewall-cmd --permanent --add-port=988/tcp && firewall-cmd --reload
# 5) 基础工具
dnf install -y net-tools bind-utils iproute vim
⚠ SELinux 修改后需重启生效；可先做完本节，重启放到 6.2 安装内核之后统一进
行。
6.2 安装 Lustre 软件（服务端 mds1/oss1）
# 1) 配置 Whamcloud 仓库
cat > /etc/yum.repos.d/whamcloud.repo <<'EOF'
[lustre-server]
name=Lustre Server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.8/el8.10/server/
enabled=1
gpgcheck=0
[lustre-client]
```


<!-- 第 19 页 -->

19
name=Lustre Client
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.8/el8.10/client/
enabled=1
gpgcheck=0
[e2fsprogs-wc]
name=Whamcloud e2fsprogs
baseurl=https://downloads.whamcloud.com/public/e2fsprogs/latest/el8/
enabled=1
gpgcheck=0
EOF
```bash
# 2) 安装依赖 EPEL
dnf install -y epel-release
# 3) 安装 Lustre 补丁内核（先查清仓库里确切的 kernel 包名/版本再装）
dnf list available 'kernel*' --enablerepo=lustre-server | grep lustre
dnf install -y kernel-4.18.0-553.82.1.el8_lustre \
kernel-devel-4.18.0-553.82.1.el8_lustre --nogpgcheck
# 4) 补丁内核设为默认启动项，并锁定内核版本
grubby --set-default /boot/vmlinuz-4.18.0-553.82.1.el8_lustre.x86_64
echo "exclude=kernel-4.18.0* kernel-core kernel-devel kernel-headers" >>
/etc/dnf/dnf.conf
# 5) 安装 Whamcloud e2fsprogs（ldiskfs 后端必需）
dnf install -y e2fsprogs --enablerepo=e2fsprogs-wc --nogpgcheck
# 6) 安装 Lustre 服务端软件
dnf install -y lustre lustre-osd-ldiskfs-mount kmod-lustre kmod-lustre-osd-ldiskfs --
nogpgcheck
# 7) 重启进入补丁内核，验证
reboot
uname -r # 期望：4.18.0-553.82.1.el8_lustre.x86_64
modprobe lustre && lsmod | grep lustre
lctl version # 期望输出 2.15.x
6.3 配置 LNet（所有节点，含客户端）
LNet 支持两种配置方式，二选一（推荐方式 1）：
# 方式 1：动态配置（2.10+ 推荐，lnetctl）
lnetctl lnet configure
lnetctl net add --net tcp0 --if ens192 # --if 填实际网卡名
lnetctl net show # 应显示 up
lctl list_nids # 期望：192.168.10.11@tcp0
# 方式 2：静态 modprobe 参数（需要持久化时）
```


<!-- 第 20 页 -->

20
cat > /etc/modprobe.d/lnet.conf <<EOF
options lnet networks=tcp0(ens192)
EOF
modprobe lustre
lctl list_nids
若使用方式 1，请把 lnetctl 命令写入 systemd 服务以保证开机自启（见 6.10）。
连通性验证（服务端之间、客户端到 MGS）：
lnetctl ping 192.168.10.11@tcp0 # 2.13 起 lctl ping 已并入 lnetctl
6.4 创建并挂载 MGS/MGT（mds1）
```bash
# MGT 很小（几 GB 足够），用一块独立盘 /dev/sdb
mkfs.lustre --fsname=lustre --mgs /dev/sdb
mkdir -p /mnt/mgt
mount -t lustre /dev/sdb /mnt/mgt
格式化输出会提示 Writing CONFIGS/mountdata，即 MGT 已初始化。
6.5 创建并挂载 MDT（mds1）
# 单 MDT 必须 --index=0；--mgsnode 填 MGS 的 NID
mkfs.lustre --fsname=lustre --mdt --index=0 \
--mgsnode=192.168.10.11@tcp0 /dev/sdc
mkdir -p /mnt/mdt
mount -t lustre /dev/sdc /mnt/mdt
6.6 创建并挂载 OST（oss1，每个 OST 一块独立盘）
# OST0：/dev/sdb
mkfs.lustre --fsname=lustre --ost --index=0 \
--mgsnode=192.168.10.11@tcp0 /dev/sdb
mkdir -p /mnt/ost0
mount -t lustre /dev/sdb /mnt/ost0
# OST1：/dev/sdc（追加 OST 只需递增 --index，无需其他节点改动）
mkfs.lustre --fsname=lustre --ost --index=1 \
--mgsnode=192.168.10.11@tcp0 /dev/sdc
mkdir -p /mnt/ost1
mount -t lustre /dev/sdc /mnt/ost1
生产 HA 环境：mkfs 时加 --servicenode=<主 NID> --servicenode=<备 NID> 声明故障切
换伙伴。mkfs.lustre 关键参数详见附录 C。
6.7 服务端验证（mds1）
lctl dl
# 期望看到多行，例如：
```


<!-- 第 21 页 -->

21
```bash
# 0 UP mgs MGS MGS 9
# 1 UP mdt MDS MDS_MDT0000 7 ...
# 2 UP osc lustre-OST0000-osc-MDT0000 ... (OSS 注册后自动出现)
# 查看 MGS 上记录的文件系统配置（应能看到 lustre-MDT0000、lustre-OST0000/0001）
lctl get_param mgs.MGS.live.*
6.8 客户端安装与挂载（client1）
# 1) 配置客户端仓库（只留 client 与 e2fsprogs 段），然后：
dnf install -y epel-release
dnf install -y lustre-client --nogpgcheck
modprobe lustre
lctl version # 客户端版本要求与服务端一致
# 2) 配置 LNet（同 6.3）
lnetctl lnet configure
lnetctl net add --net tcp0 --if ens192
lctl list_nids
# 3) 连通性测试
lnetctl ping 192.168.10.11@tcp0
# 4) 挂载文件系统
mkdir -p /mnt/lustre
mount -t lustre 192.168.10.11@tcp0:/lustre /mnt/lustre
# 5) 写入 fstab 持久化（_netdev 确保网络就绪后才挂载）
cat >> /etc/fstab <<EOF
192.168.10.11@tcp0:/lustre /mnt/lustre lustre defaults,_netdev 0 0
EOF
6.9 功能验证（client1）
# 1) 文件系统总览
lfs df -h # 应显示 MDT 与所有 OST 的容量
# 2) 查看默认布局与条带化
lfs getstripe /mnt/lustre
lfs setstripe -c 2 -S 4M /mnt/lustre/striped_test # 跨 2 个 OST、4MB 条带
lfs getstripe /mnt/lustre/striped_test # obdidx 列显示 0,1
# 3) 简单 IO 测试
dd if=/dev/zero of=/mnt/lustre/bigfile bs=1M count=1024 oflag=direct
dd if=/mnt/lustre/bigfile of=/dev/null bs=1M oflag=direct
# 用 fio 更全面：
dnf install -y fio
fio --name=test --rw=rw --bs=1M --size=512M --numjobs=4 \
--directory=/mnt/lustre --ioengine=libaio --direct=1
```


<!-- 第 22 页 -->

22
```bash
# 4) 文件在多个 OST 上真实分布
lfs getstripe /mnt/lustre/bigfile
6.10 开机自启与持久化
节点 需要持久化的内容 方法
所有节点 LNet 网络 方式 2 已随 modprobe 参数固化；方式 1 需新建
/etc/systemd/system/lnet-setup.service 执行 lnetctl
lnet configure && lnetctl net add，systemctl enable
lnet-setup
mds1 MGT/MDT 挂载 fstab：/dev/sdb /mnt/mgt lustre defaults,_netdev 0 0
（用 UUID 更稳，防设备名漂移）
oss1 OST 挂载 fstab 同上
client1 客户端挂载 fstab：...:/lustre /mnt/lustre lustre defaults,_netdev
0 0
旧式（el7） — chkconfig lnet on && chkconfig lustre on
⚠ 实测经验（见第 8.5 节）：fstab 的 _netdev 只等网卡、不等服务端 recovery
窗口。客户端挂载推荐用"先 ping MGS 直到可达再 mount"的等待式 systemd 服务，
避免整机重启后客户端"盘丢失"。
```


<!-- 第 23 页 -->

23
第 7 章 Ubuntu 24.04 部署方法论
Ubuntu 24.04（noble，内核 6.8 GA）+ Lustre 2.16.x / 2.17.x（不使用 2.15，
与 kernel 6.x 不兼容，见第 4.3 节）。本章覆盖方案选择、客户端安装（4 条路
径）、服务端源码编译；完整实测流程见第 8 章。
7.1 架构方案选择
核心选型问题是：是否让 Ubuntu 24.04 承担服务端（MGS/MDS/OSS）角色。两条方案的客
户端安装方法完全一致（7.2），区别只在服务端如何获得。
对比项 方案 A：服务端用 RHEL 家族 方案 B：全栈 Ubuntu
服务端 Rocky/RHEL 8.10 + Lustre 2.15.8 官方预编
译包（第 6 章）
Ubuntu 24.04 + 2.16.x/2.17.x 源
码编译（7.3）
客户端 Ubuntu 24.04 + 2.16.x/2.17.x 同左
互操作性 2.16 客户端 ↔ 2.15.8 服务端，官方支持 全同版本
部署时间 半天内可跑通 服务端编译数小时起步（2~6 小时）
技术栈 混合（Rocky 服务端 + Ubuntu 客户端） 全 Ubuntu 统一
一句话结论 入门首选（无需编译，踩坑最少） 有明确技术栈/源码定制需求时选
7.2 Ubuntu 24.04 客户端安装（方案 A 与 B 通用）
客户端是 patchless 的，不需要补丁内核。没有官方通用 apt 仓库，有 4 条可用路径，
任选其一（优先路径 1 或 2）。
安装前检查：
uname -r # 应为 6.8.x-generic（GA 内核，不要
HWE）
apt list --installed | grep linux-headers # 头文件版本必须与 uname -r 严格一致
gcc --version; make --version # 编译工具链（DKMS 首次编译需要）
ldd --version | head -1 # GLIBC ≥ 2.38（24.04 自带 2.39，满足）
路径 1：Azure Managed Lustre 仓库（推荐，Lustre 2.17，amd64/arm64）
来源 Microsoft 官方，仓库 https://packages.microsoft.com/repos/amlfs-<codename>/
（noble 即 amlfs-noble）。仓库提供两种包：DKMS 包（安装时按当前内核自动编译，跨内
核通用，GA 内核下推荐）与预编译 kmod 包（仅覆盖特定 -azure 内核）。
# 1) 配置软件源与密钥


<!-- 第 24 页 -->

24
sudo apt update
sudo apt install -y ca-certificates curl apt-transport-https lsb-release gnupg dpkg-
dev
source /etc/lsb-release
ARCH=$(dpkg-architecture -q DEB_BUILD_ARCH)
echo "deb [arch=${ARCH}] https://packages.microsoft.com/repos/amlfs-
${DISTRIB_CODENAME}/ ${DISTRIB_CODENAME} main" \
| sudo tee /etc/apt/sources.list.d/amlfs.list
curl -sL https://packages.microsoft.com/keys/microsoft.asc \
| gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
sudo apt update
```bash
# 2) 查看仓库中可用的 DKMS 包（版本以仓库实际为准）
apt list -a 'amlfs-lustre-client*' | grep dkms
# 3) 安装 DKMS 客户端（示例版本为撰写时的 2.17.0；以第 2 步查询结果为准）
sudo apt install -y amlfs-lustre-client-dkms-2.17.0-24-gf517bc4
# DKMS 按当前内核自动编译，首次需几分钟
路径 2：Google Cloud Artifact Registry（lustre-client-ubuntu-noble）
# 1) 安装 Artifact Registry 的 apt 传输插件
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
| sudo gpg --dearmor -o /usr/share/keyrings/google-cloud.gpg
echo 'deb [signed-by=/usr/share/keyrings/google-cloud.gpg]
http://packages.cloud.google.com/apt apt-transport-artifact-registry-stable main' \
| sudo tee /etc/apt/sources.list.d/artifact-registry.list
sudo apt update && sudo apt install -y apt-transport-artifact-registry
# 2) 配置 noble 客户端仓库
curl -fsSL https://us-apt.pkg.dev/doc/repo-signing-key.gpg \
| sudo gpg --dearmor -o /usr/share/keyrings/lustre-client.gpg
echo "deb [signed-by=/usr/share/keyrings/lustre-client.gpg] ar+https://us-
apt.pkg.dev/projects/lustre-client-modules-dkms lustre-client-ubuntu-noble main" \
| sudo tee -a /etc/apt/sources.list.d/artifact-registry.list
sudo apt update
# 3) 安装客户端
sudo apt install -y lustre-client-modules-dkms/lustre-client-ubuntu-noble
sudo apt install -y lustre-client-utils/lustre-client-ubuntu-noble
该仓库为 GCP 场景设计，ar+https:// 传输依赖 GCP VM 凭证；非 GCP 机器上如 apt
update 失败，改用路径 1 或 4。
路径 3：lustre.software 社区镜像（非官方）
镜像 Whamcloud feature 分支构建（2.16/2.17 开发线），只提供预编译模块包（无
DKMS）。⚠ 镜像未提供 GPG 密钥，必须关闭签名校验（[trusted=yes]），存在被篡改风
险，仅建议学习/实验环境使用。
```


<!-- 第 25 页 -->

25
echo 'deb [trusted=yes] https://lustre.software/mirror/latest-feature-release/latest-
feature-release/ubuntu2404/client/ ./' \
| sudo tee /etc/apt/sources.list.d/lustre-feature.list
sudo apt update
sudo apt install -y lustre-client-utils
sudo apt install -y lustre-client-modules-$(uname -r)
```bash
# 若仓库没有与 uname -r 一致的模块包，改用 DKMS 路径或源码编译
路径 4：源码编译（备选）
适用：上述仓库均无合适的包、需要特定分支/提交，或想研究客户端源码。
# 1) 构建依赖
sudo apt install -y build-essential git autoconf automake libtool \
linux-headers-$(uname -r) libreadline-dev libmount-dev libyaml-dev libnl-3-dev \
libnl-genl-3-dev libkrb5-dev libkeyutils-dev libssl-dev libhwloc-dev \
libsnmp-dev libpython3-dev flex bison pkg-config libaio-dev
# 2) 获取源码并检出目标分支（b2_16 = 2.16.x；master = 2.17.x/最新）
git clone https://github.com/lustre/lustre-release.git
cd lustre-release && git checkout b2_16
# 3) 仅构建客户端
sh autogen.sh
./configure --disable-server --with-linux=/usr/src/linux-headers-$(uname -r)
# 4) 打包（debs 绑定当前内核；dkms-debs 跨内核更省事）
make debs # 或 make dkms-debs
# 5) 安装并验证
sudo apt install -y ./debs/*.deb
sudo depmod -a && sudo modprobe lustre
lctl version
安装后验证（所有路径通用）：
sudo modprobe lustre # 无输出即为成功
lctl version # 期望 2.16.x / 2.17.x
lsmod | grep lustre
排障提示：modprobe 报 Invalid module format 或 Unknown symbol，绝大多数原因是内
核头文件与运行内核不一致（见 7.1 检查项与第 13 章排障表）。
7.3 Ubuntu 24.04 服务端源码编译（MGS/MDS/OSS 角色）
与客户端不同，Ubuntu 24.04 没有官方预编译服务端 .deb，服务端角色必须从源码编
译，这是 Ubuntu 与 RHEL 系的最大差异。编译耗时约 2~6 小时（视 CPU/内存），属
正常现象。核心链路：确认内核/头文件 → 安装编译依赖 → 选定后端（ZFS 或
```


<!-- 第 26 页 -->

26
ldiskfs）→ git clone → autogen.sh → configure → make debs → dpkg -i → depmod
→ modprobe 验证。
（1）确认内核与头文件（内核头文件必须与正在运行的内核严格一致）：
uname -r # 应为 6.8.0-XX-generic
sudo apt install -y linux-headers-$(uname -r)
apt list --installed | grep linux-headers # 复核版本一致
ls -l /lib/modules/$(uname -r)/build # 应指向 headers 目录
```bash
# 锁定内核（强烈建议）
sudo apt-mark hold linux-image-generic linux-headers-generic
（2）安装编译依赖：
sudo apt install -y build-essential gcc make flex bison pkg-config \
zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
python3-dev dpkg-dev
依赖 用途
build-essential / gcc / make 编译工具链（Ubuntu 24.04 默认 gcc-13 与 6.8
内核一致，保持默认）
flex / bison 生成解析器代码
pkg-config / libtool / autoconf autogen.sh 与 configure 需要
zlib1g-dev / libssl-dev / libkeyutils-dev
/ libkrb5-dev / libreadline-dev
各类库绑定
libmount-dev / libyaml-dev / libnl-3-dev
/ libnl-genl-3-dev
挂载与 LNet 用户态配置
swig / python3-dev Python 绑定
dpkg-dev 提供 dpkg-buildpackage，make debs 打包必需
（3）后端选择（二选一，编译前必须确定）：
对比项 路线 A：ZFS 后端 路线 B：ldiskfs（patchless）
内核补丁 完全不需要（ZFS 模块来自 zfs-
dkms）
不需要补丁内核，但需要 LU-18010 的 ext4
源码链接法
额外组件 Ubuntu 自带 ZFS 2.2.x Whamcloud 定制 e2fsprogs + linux-
source-6.8.0
```


<!-- 第 27 页 -->

27
target 设备写
法
<zpool>/<dataset> 块设备路径（如 /dev/sdc）
configure 参
数
--with-zfs --disable-ldiskfs 默认（ldiskfs 自动启用）
结论 推荐，免补丁最省心 与 RHEL 系命令最接近，但多做两步准备
路线 A：ZFS 后端（推荐）
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs
zpool version # 查看已加载的 ZFS 版本
ls /usr/src | grep zfs # 期望形如 /usr/src/zfs-2.2.x（configure 探测用）
路线 B：ldiskfs（LU-18010 ext4 链接法）
ldiskfs 模块编译需要 ext4 内核源码，而 Ubuntu 的 linux-headers 只有编译产物、没
有源码。官方 JIRA LU-18010 给出已验证的简化做法：把内核源码包里的 fs/ext4 目录符
号链接进 headers：
```bash
# (a) 编译安装 Whamcloud 定制 e2fsprogs（版本号带 wc 后缀，ldiskfs 必需）
git clone https://review.whamcloud.com/tools/e2fsprogs
cd e2fsprogs && git checkout master-lustre
sed -i 's/ext2_types-wrapper.h$//g' lib/ext2fs/Makefile.in
./configure
dpkg-buildpackage -b -us -uc && cd ..
sudo dpkg -i *.deb; sudo dpkg -i *.deb # 有相互依赖，重复执行一次
e2fsprogs -V # 确认版本带 wc 后缀
# (b) 把内核源码的 ext4 目录链接进 headers
sudo apt install -y linux-source-6.8.0
cd /usr/src/linux-source-6.8.0 && sudo tar xjf linux-source-6.8.0.tar.bz2
KVER=$(uname -r)
sudo mv /usr/src/linux-headers-${KVER}/fs/ext4 /usr/src/linux-headers-
${KVER}/fs/ext4.orig
sudo ln -s /usr/src/linux-source-6.8.0/linux-source-6.8.0/fs/ext4 \
/usr/src/linux-headers-${KVER}/fs/ext4
⚠ 定制 e2fsprogs 会覆盖系统原生版本，覆盖前建议 dpkg -l | grep e2fsprogs 记
录原版本便于回退。编译完成后可删除链接并还原 ext4.orig。
（4）编译安装（make debs）：
# 1) 获取源码并检出目标分支
git clone https://github.com/lustre/lustre-release.git
cd lustre-release
git checkout b2_16 # 2.16.x 稳定线；或 master（2.17 开发线）
```


<!-- 第 28 页 -->

28
```bash
# 2) 生成 configure
sh autogen.sh
# 3) configure —— 按后端二选一
./configure --enable-server --with-zfs --disable-ldiskfs \
--with-linux=/usr/src/linux-headers-$(uname -r)
# 路线 B：./configure --enable-server --with-linux=/usr/src/linux-headers-$(uname -r)
# （已装 zfs 但不用时追加 --without-zfs）
# 4) 打包（耗时较长属正常）
make debs -j$(nproc)
ls debs/
# 5) 安装并验证
sudo dpkg -i debs/*.deb
sudo depmod -a
sudo modprobe libcfs && sudo modprobe lustre
lctl version # 期望 2.16.x / 2.17.x
要点：服务端所有节点必须安装同一套 debs、使用同一内核，否则目标注册/挂载会出现
版本不一致；换内核后必须重编。常见编译报错见第 13.2 节。
```


<!-- 第 29 页 -->

29
第 8 章 Ubuntu 3 节点集群实测部署记录
本章记录 2026-08-20 在三台 Ubuntu 24.04 虚拟机上真实完成的一次 Lustre
2.17.0 全栈部署：从环境探查、内核修正、服务端源码编译，到目标创建、客户端挂
载、开机自启与重启自愈验证的全部实际操作过程与结果。方法论与原理见第 7 章，两章
配合阅读。
8.1 部署结果摘要
 集群构成：3 台 Ubuntu 24.04.2 LTS 虚拟机，Lustre 2.17.0（服务端源码编译 +
客户端 Azure DKMS），存储后端 ZFS 2.2.2，网络 LNet tcp（1GbE）；
 挂载：客户端 client1 上 192.168.12.220@tcp:/lustre → /mnt/lustre；
 容量：MDT0000 23.5G + OST0000/OST0001 各 47.6G = 合计 95.2G；
 验证通过：三机 LNet 互 ping 通；lfs setstripe -c 2 文件确认横跨
OST0000/OST0001；dd 写 512M 约 114 MB/s（1GbE 带宽上限附近）；
 开机自愈：三台重启后 ZFS 池自动导入、Lustre 目标自动挂载、客户端自动挂载恢
复；
 关键前置修正：三台初始内核为 7.0.0-29-generic（HWE 线），与 Lustre 2.17 /
ZFS 2.2.2 支持的内核线不符，已切回 6.8.0-138-generic（GA）并锁定。
节点 主机名 IP 登录用户 角色
mds1 mds1 192.168.12.220 user MGS + MDS（MDT0000）
oss1 oss1 192.168.12.221 user2 OSS（OST0000 / OST0001）
client1 client1 192.168.12.222 user1 Client
8.2 实际环境信息
节点 vCPU 内存 系统盘 数据盘
mds1 4 3.8G sda 50G（ext4，/） sdb 5G = MGT；sdc 25G = MDT
oss1 4 3.8G sda 50G（ext4，/） sdb 50G = OST0；sdc 50G =
OST1
client1 4 3.8G sda 40G（ext4，/） 无
 数据盘均为裸盘（未分区、未格式化、未挂载），直接交给 ZFS 建池；
 虚拟化平台：VMware；网卡 ens33；静态 IP，网关 192.168.12.254；时区
Asia/Shanghai。
组件 版本
发行版 Ubuntu 24.04.2 LTS（noble）
内核（最终） 6.8.0-138-generic（GA，apt-mark hold 锁定）
Lustre 服务端 2.17.0（git b2_17 分支，commit 4b0407a，make debs 编译）
Lustre 客户端 2.17.0（Azure DKMS 包 amlfs-lustre-client-dkms-2.17.0-24-gf517bc4）
ZFS 2.2.2-0ubuntu9.4（zfsutils-linux + zfs-dkms + libzfslinux-dev）


<!-- 第 30 页 -->

30
gcc 13（Ubuntu 24.04 默认）
8.3 部署操作全过程
以下步骤按实际执行顺序记录；<节点> 标注该步在哪台机器上执行；"三台"=
mds1/oss1/client1。
（1）环境探查（部署前，三台）
ssh user@192.168.12.220
cat /etc/os-release | head -3
uname -r # 实测为 7.0.0-29-generic（HWE 线，需修正）
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
id; sudo -n true && echo NOPASS || echo NEEDS_PASS
（2）内核修正：切回 6.8 GA 并锁定（关键前置，三台执行）
```bash
# 1) 安装 GA 内核与头文件（6.8 线）
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic
ls /boot/vmlinuz-* # 确认出现 vmlinuz-6.8.0-138-generic
# 2) 设置 GRUB 默认启动 6.8（菜单项写法以 update-grub 实际输出为准）
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu,
with Linux 6.8.0-138-generic"/' /etc/default/grub
sudo update-grub
# 3) 重启并确认进入 6.8
sudo reboot
uname -r # 期望 6.8.0-138-generic
# 4) 锁定内核
sudo apt-mark hold linux-image-generic linux-headers-generic
# 5) 移除 HWE 7.0 内核（必须！否则 zfs-dkms 会为 7.0 构建失败卡死 dpkg）
sudo apt purge -y linux-image-7.0.0-29-generic linux-headers-7.0.0-29-generic \
linux-image-generic-hwe-24.04 linux-headers-generic-hwe-24.04 \
linux-modules-7.0.0-29-generic
⚠ 实测踩坑：若先装 zfs-dkms 再 purge 7.0，安装会卡在"为 7.0 内核构建 ZFS 模
块"。正确顺序是先切 6.8 并 purge 7.0，再装 ZFS。
（3）基础环境准备（三台执行）
# 1) 主机名（各台分别设置）
sudo hostnamectl set-hostname mds1
# 2) /etc/hosts 三机映射（三台内容一致；实测需删除默认的 127.0.1.1 行避免歧义）
```


<!-- 第 31 页 -->

31
sudo tee -a /etc/hosts <<'EOF'
192.168.12.220 mds1
192.168.12.221 oss1
192.168.12.222 client1
EOF
sudo sed -i '/^127.0.1.1 /d' /etc/hosts
getent hosts mds1 oss1 client1
```bash
# 3) 时间同步（Lustre 对时钟敏感，防客户端被 evict）
sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc sources -v
# 4) 公共工具 + 与运行内核严格一致的头文件
sudo apt install -y git curl wget ca-certificates gnupg lsb-release software-
properties-common \
build-essential gcc make linux-headers-$(uname -r)
（4）服务端依赖与 ZFS（mds1、oss1）
# 1) 编译依赖（实测需补 libzfslinux-dev / module-assistant / debhelper / quilt）
sudo apt install -y build-essential gcc make flex bison pkg-config \
zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
python3-dev dpkg-dev \
libzfslinux-dev module-assistant debhelper quilt
# 2) ZFS（zfs-dkms 按当前 6.8 内核编译模块）
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs && echo ZFS_LOADED
zpool version # 期望 zfs-2.2.2-0ubuntu9.4
ls /usr/src | grep zfs # 期望 /usr/src/zfs-2.2.2
（5）服务端源码编译（mds1、oss1，约 40 分钟）
GitHub 直连在本网络很慢，走代理：
# 1) 克隆并检出 b2_17 分支
export http_proxy=http://192.168.12.187:7897 https_proxy=http://192.168.12.187:7897
git clone --depth 1 https://github.com/lustre/lustre-release.git
cd ~/lustre-release
git fetch --depth 1 origin b2_17
git checkout -b b2_17 FETCH_HEAD
git log --oneline -1 # 期望 4b0407a New release 2.17.0
# 2) 生成 configure 并配置（ZFS 后端、禁用 ldiskfs）
sh autogen.sh
./configure --enable-server --with-zfs --disable-ldiskfs \
--with-linux=/usr/src/linux-headers-$(uname -r)
# 成功标志：结尾 "Type 'make' to build Lustre." + "checking if ZFS has ... yes"
```


<!-- 第 32 页 -->

32
```bash
# 3) 编译打包（4 vCPU/3.8G 用 -j4）
nohup make debs -j4 > /tmp/lustre_make.log 2>&1 &
tail -f /tmp/lustre_make.log # 轮询进度
ls ~/lustre-release/debs/*.deb # 产出 7 个 .deb（含 lustre-server-modules-6.8.0-
138-generic）
实测说明：osd-zfs 内核模块内含于 lustre-server-modules-<内核> 包，没有独立的
osd-zfs .deb，属正常。
（6）安装服务端 .deb 并加载模块（mds1、oss1）
cd ~/lustre-release/debs
sudo dpkg -i *.deb
sudo depmod -a
sudo modprobe libcfs && echo LIBCFS_OK
sudo modprobe lustre && echo LUSTRE_OK
lctl version # 期望 2.17.0
（7）LNet 配置（三台执行）
# 本机网卡为 ens33；2.17 下 NID 显示为 192.168.12.x@tcp
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lnetctl net show
sudo lctl list_nids # 期望 192.168.12.220@tcp（mds1）等
# 连通性验证（三机互 ping）
sudo lnetctl ping 192.168.12.221@tcp # mds1 上 ping oss1
sudo lnetctl ping 192.168.12.220@tcp # client1 上 ping mds1（客户端必须能到 MGS）
实测差异：方案示例写 @tcp0，2.17 实际 lctl list_nids 显示 @tcp，二者对应同一网
络，互 ping 用 @tcp 即可。
（8）创建并挂载 MGS/MDT（mds1）
# 1) 建池：/dev/sdb = MGT，/dev/sdc = MDT
sudo zpool create lustre-mgt /dev/sdb
sudo zpool create lustre-mdt /dev/sdc
sudo zpool list
# 2) mkfs：先 MGS 再 MDT；--mgsnode 指向 mds1 自身 NID
sudo mkfs.lustre --fsname=lustre --mgs --backfstype=zfs lustre-mgt/mgt
sudo mkfs.lustre --fsname=lustre --mdt --index=0 \
--mgsnode=192.168.12.220@tcp --backfstype=zfs lustre-mdt/mdt
# 3) 挂载
sudo mkdir -p /mnt/mgt /mnt/mdt
```


<!-- 第 33 页 -->

33
sudo mount -t lustre lustre-mgt/mgt /mnt/mgt
sudo mount -t lustre lustre-mdt/mdt /mnt/mdt
mount | grep lustre
（9）创建并挂载 OST（oss1）
```bash
# 1) 建池：/dev/sdb = OST0，/dev/sdc = OST1
sudo zpool create lustre-ost0 /dev/sdb
sudo zpool create lustre-ost1 /dev/sdc
# 2) mkfs：--index 从 0 递增；--mgsnode 指向 mds1
sudo mkfs.lustre --fsname=lustre --ost --index=0 \
--mgsnode=192.168.12.220@tcp --backfstype=zfs lustre-ost0/ost
sudo mkfs.lustre --fsname=lustre --ost --index=1 \
--mgsnode=192.168.12.220@tcp --backfstype=zfs lustre-ost1/ost
# 3) 挂载
sudo mkdir -p /mnt/ost0 /mnt/ost1
sudo mount -t lustre lustre-ost0/ost /mnt/ost0
sudo mount -t lustre lustre-ost1/ost /mnt/ost1
mount | grep lustre
（10）服务端目标验证（mds1）
sudo lctl dl # 期望 MGS / MDT0000 / 相关 OSC 行均 UP
sudo lctl get_param mgs.MGS.live.* # 期望列出 lustre-MDT0000 / OST0000 / OST0001
实测输出摘要：lctl dl 显示 MGS、lustre-MDT0000 等 12 行 UP；mgs.MGS.live.lustre
下可见 lustre-MDT0000、lustre-OST0000、lustre-OST0001 三项——全部目标注册成功。
（11）客户端安装与挂载（client1）
# 1) 配置 Azure Managed Lustre 仓库（DKMS 路径，同第 7.2 路径 1）
sudo apt install -y ca-certificates curl apt-transport-https lsb-release gnupg dpkg-
dev
source /etc/lsb-release
ARCH=$(dpkg-architecture -q DEB_BUILD_ARCH)
echo "deb [arch=${ARCH}] https://packages.microsoft.com/repos/amlfs-
${DISTRIB_CODENAME}/ ${DISTRIB_CODENAME} main" \
| sudo tee /etc/apt/sources.list.d/amlfs.list
curl -sL https://packages.microsoft.com/keys/microsoft.asc \
| gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
sudo apt update
# 2) 安装 DKMS 包（实测包名 amlfs-lustre-client-dkms-2.17.0-24-gf517bc4）
sudo apt install -y amlfs-lustre-client-dkms-2.17.0-24-gf517bc4
# 3) 加载模块并验证（必须 sudo）
sudo modprobe lustre
```


<!-- 第 34 页 -->

34
lctl version # 期望 2.17.0_24_gf517bc4
lsmod | grep -E 'lustre|mdc|lov'
```bash
# 4) LNet（同本节第 7 步）+ 挂载
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lnetctl ping 192.168.12.220@tcp # 先确认 MGS 可达
sudo mkdir -p /mnt/lustre
sudo mount -t lustre 192.168.12.220@tcp:/lustre /mnt/lustre
mount | grep lustre
实测差异：普通用户 modprobe lustre 报 Operation not permitted，必须 sudo。
（12）功能验证（client1）
# 1) 容量总览：应看到 MDT0000 + OST0000 + OST0001
lfs df -h /mnt/lustre
# 2) 条带化：目录设默认布局跨 2 OST，创建文件验证分布
sudo mkdir -p /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test/striped2
lfs getstripe -v /mnt/lustre/test/striped2 # lmm_stripe_count: 2；obdidx 含 0 和 1
# 3) IO 测试
dd if=/dev/zero of=/mnt/lustre/test/bigfile bs=1M count=512 oflag=direct
conv=fdatasync
实测结果：lfs df -h 显示 MDT0000 23.5G、OST0000 47.6G、OST0001 47.6G，
filesystem_summary 95.2G；lfs getstripe -v 显示 stripe_count=2，对象落在
obdidx 0 与 1——条带化生效；dd 写 512M：114 MB/s（1GbE 上限附近）。
8.4 开机自启配置与重启自愈验证
# 1) fstab：服务端目标用 fstab + _netdev（客户端挂载例外，见 8.5）
# mds1 追加：
# lustre-mgt/mgt /mnt/mgt lustre defaults,_netdev 0 0
# lustre-mdt/mdt /mnt/mdt lustre defaults,_netdev 0 0
# oss1 追加：
# lustre-ost0/ost /mnt/ost0 lustre defaults,_netdev 0 0
# lustre-ost1/ost /mnt/ost1 lustre defaults,_netdev 0 0
# 2) ZFS 池写入 cachefile（开机自动导入）
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-mgt lustre-mdt # mds1
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-ost0 lustre-ost1 # oss1
# 3) LNet 开机自启（三台，--if ens33）
sudo tee /etc/systemd/system/lnet-setup.service >/dev/null <<'EOF'
[Unit]
```


<!-- 第 35 页 -->

35
Description=LNet dynamic setup
After=network-online.target
Wants=network-online.target
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/lnetctl lnet configure
ExecStart=/usr/sbin/lnetctl net add --net tcp0 --if ens33
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now lnet-setup
重启自愈验证（按 mds1 → oss1 → client1 顺序依次重启，每台等约 75~90 秒）：
节点 内核 ZFS 池 Lustre 目标 挂载
mds1 6.8.0-138-
generic
自动导入 12 个 UP /mnt/mgt、/mnt/mdt 自动挂载
oss1 6.8.0-138-
generic
自动导入 8 个 UP /mnt/ost0、/mnt/ost1 自动挂载
client1 6.8.0-138-
generic
— 6 个客户端目标 /mnt/lustre 自动挂载，lfs df -h
正常
8.5 故障复盘：三台同时重启后客户端挂载丢失（已解决）
现象：三台虚拟机同时重启后，client1 上 mount | grep lustre 无输出、ls
/mnt/lustre 报目录不存在，看起来"挂载的盘没了"；mds1/oss1 服务端目标均正常。
根因（两层）：
4. MGS/MDT 的 recovery 窗口期：mds1 重启后 MDT 进入 RECOVERING 状态（默认最长
300s），此期间拒绝客户端连接（内核日志 mds_connect to node ... failed: rc = -
16，即 EBUSY）；
5. fstab _netdev 只等网卡、不等服务：客户端开机早期即尝试挂载 → 被 recovery 窗
口拒绝 → systemd 挂载单元超时 → 失败后不再自动重试，表现为"盘没了"。
排查命令：
```bash
# client1：确认服务端是否可达、目标是否注册
sudo lnetctl ping 192.168.12.220@tcp # 通 → 网络层 OK
mount | grep lustre # 应为空（未挂载）
# mds1：确认 MDT recovery 状态
sudo lctl get_param mdt.lustre-MDT0000.recovery_status
# 期望 status: COMPLETE（RECOVERING 期间客户端连不上）
# client1 内核日志定位 rc=-16 来源
sudo journalctl -k | grep -i lustre | tail
```


<!-- 第 36 页 -->

36
解决方案（已实施并验证）：不用 fstab 挂载客户端，改为自定义 systemd 服务——先循环
ping MGS 直到可达（最多 300s），再执行 mount：
```bash
# client1：
# 1) 从 /etc/fstab 删除 192.168.12.220@tcp:/lustre 行
# 2) 创建 /etc/systemd/system/lustre-client-mount.service：
sudo tee /etc/systemd/system/lustre-client-mount.service >/dev/null <<'EOF'
[Unit]
Description=Mount Lustre client /mnt/lustre (waits for MGS)
After=network-online.target lnet-setup.service
Wants=network-online.target
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'for i in $(seq 1 30); do /usr/sbin/lnetctl ping
192.168.12.220@tcp >/dev/null 2>&1 && break; sleep 10; done'
ExecStart=/bin/mount -t lustre 192.168.12.220@tcp:/lustre /mnt/lustre
ExecStartPost=/bin/sleep 2
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now lustre-client-mount
sudo systemctl is-active lustre-client-mount # 期望 active
lfs df -h /mnt/lustre
验证结果：三台同时重启后，client1 的 lustre-client-mount 服务等待 MGS recovery
完成后自动挂载成功。
经验总结：Lustre 客户端挂载不建议依赖 fstab 的 _netdev（它不等服务端
recovery），生产环境应使用"等待 MGS 可达再挂载"的 systemd 服务（或 x-
systemd.automount 懒挂载），避免整机同时断电重启后客户端"盘丢失"的误判。
8.6 验证结果汇总
验证项 命令 结果
内核 uname -r（三台） 6.8.0-138-generic（GA，已 hold）
服务端版本 lctl version（mds1/oss1） 2.17.0
客户端版本 lctl version（client1） 2.17.0_24_gf517bc4
LNet sudo lctl list_nids 192.168.12.220/221/222@tcp
LNet 连通 sudo lnetctl ping 三机两两互通
目标注册 sudo lctl get_param
mgs.MGS.live.*
MDT0000 + OST0000 + OST0001 全部
注册
挂载 mount \ grep lustre（client1） 192.168.12.
220@tcp:/lu
stre on
/mnt/lustre
容量 lfs df -h MDT 23.5G + OST×2 47.6G×2 =
95.2G
```


<!-- 第 37 页 -->

37
条带化 lfs getstripe -v stripe_count=2，横跨
OST0000/0001
IO dd 写 512M 114 MB/s
重启自愈 三台依次重启 全部自动恢复，无需干预


<!-- 第 38 页 -->

38
第 9 章 单机 All-in-One 部署
Lustre 的组件（MGS/MDS/OSS/Client）是逻辑角色而非物理要求，官方 Quick Start
本身就支持单机（All-in-One）部署。适合功能验证、命令学习、给同事演示；不适合性
能测试与 HA 演练（单机无法体现并行扩展，也没有故障隔离）。
9.1 单机部署原理与适用边界
 一个节点同时承担 MGS、MDS、OSS、Client 四个角色，各自加载对应内核模块即可共
存；
 数据仍走本机 LNet 回环，条带化、锁、挂载等功能与多节点完全一致；
 区别仅在：没有网络延迟/带宽体现，看不到横向扩展的聚合吞吐。
适用：功能验证、命令学习、跑通全流程后再上多节点。不适用：性能基准（单机带宽就是上
限）、HA/故障切换演练、多客户端并发锁争用测试。
9.2 实施路径
单机部署推荐按"VM 验证 → 实例化落地 → 物理机生产档"三步推进：
路径 场景 配置档位 状态
VM 验证（9.3~9.5） 先在虚拟机上验证"单机
能否部署"
实验档：4 vCPU / 8G
/ MGT 1~2G + MDT
10~20G + OST 20G×2
投入最小、最安全
实例化落地（9.6） 在具体机器上照做 按 VM 档实例化 已实测完成
（192.168.12.160）
物理机生产档（9.7） 借物理测试机部署 生产档：见 5.7 方案
命令三份完全同构，差异仅在配置档位与 IP/盘符。VM 验证的边界：性能数字无参考价
值（虚拟磁盘/网络），验证目的是流程可行性，不是性能。
9.3 VM 实验档配置
项 建议 说明
平台 VMware Workstation / KVM / VirtualBox 与第 8 章环境一致用 VMware 亦可
CPU 4 vCPU（2 vCPU 也能跑） 编译 -j4 更快
内存 8 GB（最小 4 GB） ZFS ARC 占一半；4 GB 编译+运行
偏紧
系统盘 50 GB（thin 可） 系统 + 编译产物（源码约 128M +
debs）
数据盘 MGT 1~2 GB + MDT 10~20 GB + OST 20
GB×2，一 target 一块独立虚拟盘
每类 target 独立裸盘，不要和系统
盘混用
磁盘控制器 VMware 用 PVSCSI；KVM 用 virtio-scsi；
VBox 用 SATA
见 5.4
VM 创建注意点：绝对不要对运行中的 Lustre VM 做整机快照/回滚（见 5.4）；安装
Ubuntu 24.04 后若内核是 7.0 HWE 线，按 9.4 步骤 1 切回 6.8 GA 并锁定。


<!-- 第 39 页 -->

39
9.4 通用部署流程（8 步，全在一台机器）
以下假设：登录用户 user，网卡 ens33，数据盘 /dev/sdb(MGT)、/dev/sdc(MDT)、
/dev/sdd、/dev/sde(OST×2)，本机 IP 以 <本机 IP> 表示。按实际替换。
步骤 1：环境探查与内核修正
cat /etc/os-release | head -3
uname -r # 若为 7.0.0-xx-generic（HWE 线），执行下方内核修正
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
```bash
# 内核修正（仅当 uname -r 为 7.0 HWE；与第 8.3 节步骤 2 相同）
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu,
with Linux 6.8.0-138-generic"/' /etc/default/grub
sudo update-grub
sudo reboot
uname -r # 期望 6.8.0-138-generic
sudo apt-mark hold linux-image-generic linux-headers-generic
sudo apt purge -y linux-image-7.0.0-29-generic linux-headers-7.0.0-29-generic \
linux-image-generic-hwe-24.04 linux-headers-generic-hwe-24.04 \
linux-modules-7.0.0-29-generic
步骤 2：基础环境
sudo hostnamectl set-hostname lustre-single
sudo apt install -y chrony
sudo systemctl enable --now chrony
sudo apt install -y git curl wget ca-certificates gnupg lsb-release software-
properties-common \
build-essential gcc make linux-headers-$(uname -r)
# 关闭/调低 swap（防超时 evict）
sudo swapoff -a && sudo sed -i '/ swap /s/^/#/' /etc/fstab
步骤 3：ZFS 与编译依赖
sudo apt install -y build-essential gcc make flex bison pkg-config \
zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
python3-dev dpkg-dev \
libzfslinux-dev module-assistant debhelper quilt
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs && echo ZFS_LOADED
zpool version # 期望 zfs-2.2.2-0ubuntu9.4
步骤 4：服务端源码编译（约 40 分钟，后台）
# GitHub 直连慢时走代理
export http_proxy=http://192.168.12.187:7897 https_proxy=http://192.168.12.187:7897
```


<!-- 第 40 页 -->

40
git clone --depth 1 https://github.com/lustre/lustre-release.git
cd ~/lustre-release
git fetch --depth 1 origin b2_17
git checkout -b b2_17 FETCH_HEAD
git log --oneline -1 # 期望 4b0407a New release 2.17.0
sh autogen.sh
./configure --enable-server --with-zfs --disable-ldiskfs \
--with-linux=/usr/src/linux-headers-$(uname -r)
```bash
# 成功标志：结尾 "Type 'make' to build Lustre." + "checking if ZFS has ... yes"
nohup make debs -j4 > /tmp/lustre_make.log 2>&1 &
tail -f /tmp/lustre_make.log # 轮询：产出 7 个 .deb
ls ~/lustre-release/debs/*.deb
步骤 5：安装模块并配置 LNet
cd ~/lustre-release/debs
sudo dpkg -i *.deb
sudo depmod -a
sudo modprobe libcfs && echo LIBCFS_OK
sudo modprobe lustre && echo LUSTRE_OK
lctl version # 期望 2.17.0
# LNet：本机网卡 ens33；单机 NID 即本机 IP
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lctl list_nids # 期望 <本机 IP>@tcp
单机部署服务端 deb 已包含客户端功能（llite/osc 模块），无需再装 Azure DKMS
客户端包。
步骤 6：建池 + 创建 MGS/MDT/OST
# 1) 四个池（MGT/MDT/OST0/OST1 各一，单盘单池）
sudo zpool create lustre-mgt /dev/sdb
sudo zpool create lustre-mdt /dev/sdc
sudo zpool create lustre-ost0 /dev/sdd
sudo zpool create lustre-ost1 /dev/sde
sudo zpool list # 四个池在线
# 2) mkfs：MGS → MDT(index=0) → OST(index=0,1)；--mgsnode 指向本机 NID
sudo mkfs.lustre --fsname=lustre --mgs --backfstype=zfs lustre-mgt/mgt
sudo mkfs.lustre --fsname=lustre --mdt --index=0 \
--mgsnode=<本机 IP>@tcp --backfstype=zfs lustre-mdt/mdt
sudo mkfs.lustre --fsname=lustre --ost --index=0 \
--mgsnode=<本机 IP>@tcp --backfstype=zfs lustre-ost0/ost
sudo mkfs.lustre --fsname=lustre --ost --index=1 \
--mgsnode=<本机 IP>@tcp --backfstype=zfs lustre-ost1/ost
```


<!-- 第 41 页 -->

41
```bash
# 3) 挂载服务端目标
sudo mkdir -p /mnt/mgt /mnt/mdt /mnt/ost0 /mnt/ost1
sudo mount -t lustre lustre-mgt/mgt /mnt/mgt
sudo mount -t lustre lustre-mdt/mdt /mnt/mdt
sudo mount -t lustre lustre-ost0/ost /mnt/ost0
sudo mount -t lustre lustre-ost1/ost /mnt/ost1
mount | grep lustre # 四行目标挂载
# 4) 服务端注册验证
sudo lctl dl # MGS / MDT0000 / OST0000 / OST0001 均 UP
sudo lctl get_param mgs.MGS.live.*
只有单块大盘时：分区后各池建在不同分区上（如 /dev/nvme0n1p1），mkfs/mount 命令
不变。
步骤 7：客户端挂载（本机即是客户端）
sudo mkdir -p /mnt/lustre
sudo mount -t lustre <本机 IP>@tcp:/lustre /mnt/lustre
mount | grep lustre # 出现 <本机 IP>@tcp:/lustre on /mnt/lustre
lfs df -h /mnt/lustre # MDT0000 + OST0000 + OST0001
步骤 8：开机自启
# 1) fstab：服务端目标 + ZFS cachefile（客户端挂载不用 fstab，见 8.5 经验）
# 追加：
# lustre-mgt/mgt /mnt/mgt lustre defaults,_netdev 0 0
# lustre-mdt/mdt /mnt/mdt lustre defaults,_netdev 0 0
# lustre-ost0/ost /mnt/ost0 lustre defaults,_netdev 0 0
# lustre-ost1/ost /mnt/ost1 lustre defaults,_netdev 0 0
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-mgt lustre-mdt lustre-ost0
lustre-ost1
# 2) LNet 自启（含 modprobe lnet 前置，避免重启后 failed）
sudo tee /etc/systemd/system/lnet-setup.service >/dev/null <<'EOF'
[Unit]
Description=LNet dynamic setup
After=network-online.target
Wants=network-online.target
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/usr/sbin/modprobe lnet
ExecStart=/usr/sbin/lnetctl lnet configure
ExecStart=/usr/sbin/lnetctl net add --net tcp0 --if ens33
[Install]
```


<!-- 第 42 页 -->

42
WantedBy=multi-user.target
EOF
sudo systemctl enable --now lnet-setup
```bash
# 3) 客户端挂载用等待式 systemd（fstab _netdev 不等 MGS recovery，重启会丢挂载）
sudo tee /etc/systemd/system/lustre-client-mount.service >/dev/null <<'EOF'
[Unit]
Description=Mount Lustre client /mnt/lustre (waits for MGS)
After=network-online.target lnet-setup.service
Wants=network-online.target
[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'for i in $(seq 1 30); do /usr/sbin/lnetctl ping <本机
IP>@tcp >/dev/null 2>&1 && break; sleep 10; done'
ExecStart=/bin/mount -t lustre <本机 IP>@tcp:/lustre /mnt/lustre
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now lustre-client-mount
9.5 部署成功判定清单
逐项勾选，全部通过即"单机部署验证通过"。这也是后续物理机部署的验收标准（命令相
同）。
# 验证项 命令 通过标准
1 内核正确 uname -r 6.8.0-xxx-
generic（GA，已
hold）
2 服务端版本 lctl version 2.17.0
3 ZFS 加载 zpool version zfs-2.2.x
4 LNet 就绪 sudo lctl list_nids 本机 IP@tcp
5 目标全部注册 sudo lctl get_param
mgs.MGS.live.*
MDT0000 +
OST0000 +
OST0001 全部列出
6 客户端挂载 mount \ grep lustre <本机
IP>@tcp:/lustre
on /mnt/lustre
7 容量正确 lfs df -h /mnt/lustre MDT + 2×OST，无
异常 0 值
8 条带化生效 lfs getstripe -v <测试文件> obdidx 含 0 和
1
9 IO 可写 dd if=/dev/zero of=...
bs=1M count=512
oflag=direct conv=fdatasync
无报错，字节数正
确
10 重启自愈 sudo reboot 后 lfs df -h 自动恢复，客户端
服务 active
```


<!-- 第 43 页 -->

43
功能验证（第 8 项前置）：
sudo mkdir -p /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test
sudo dd if=/dev/zero of=/mnt/lustre/test/bigfile bs=1M count=512 oflag=direct
conv=fdatasync
lfs getstripe -v /mnt/lustre/test/bigfile
9.6 实例化案例：192.168.12.160（2026-08-26 实测完成）
本节是 9.4 通用流程在具体机器 192.168.12.160 上的落地记录：所有 IP、用户名、
盘符均为实测值。登录用户 ps（sudo 密码 1），网卡 ens33。
实测环境：Ubuntu 24.04.4 LTS；初始内核 7.0.0-30-generic（HWE，已切回 6.8.0-
138-generic 并锁定）；4 vCPU / 7.7 GiB。
实测结果：
验证项 实测结果
Lustre 服务端 2.17.0（b2_17 / commit 4b0407a，make debs 编译，7 个 .deb）
ZFS 2.2.2-0ubuntu9.4
LNet 192.168.12.160@tcp（ens33，状态 up）
目标 MGS / MDT0000 / OST0000 / OST0001 全部注册，lctl dl 24 项 UP
客户端 192.168.12.160@tcp:/lustre → /mnt/lustre（等待式 systemd 自启，
active）
容量 MDT 18.7G + OST0000 38.0G + OST0001 28.4G = 66.3G
条带化 lfs setstripe -c 2 -S 4M 生效，文件横跨 OST0000/0001
IO dd 写 512M = 167 MB/s（VM 本机回环）
重启自愈 重启后 zpool 自动导入、目标自动挂载、客户端自动恢复
⚠ 实测发现：磁盘设备名漂移（重要，已修复）
部署前在 7.0 内核下 lsblk 探测为 sdb=20G / sdc=2G / sdd=40G / sde=30G；切回
6.8 内核重启后设备名发生交换（sdb↔sdc、sdd↔sde）。首轮建池因此错位（MGT 拿到
20G 盘、MDT 只有 2G 盘）。已销毁四池后按当前设备名重建修复。
? 教训：① 内核切换/重启后 /dev/sdX 设备名可能漂移，建池前务必重新 lsblk 确认
当前映射；② 生产环境应使用 /dev/disk/by-id/（或 by-uuid/by-path）建池，规避漂
移；③ 池创建后 vdev 绑定设备内部 ID，后续重启不受设备名变化影响，仅"创建时"需
注意。
9.7 物理机生产档要点
在物理测试机上部署时，按 5.7 生产档配置准备硬件（8~16 核 / 32~64G ECC / NVMe
MDT / OST 带 mirror/RAID-Z 冗余 / 10GbE+），命令与 9.4 完全相同，另注意：
 建池用 /dev/disk/by-id/ 路径规避设备名漂移；


<!-- 第 44 页 -->

44
 OST 后端必须带冗余（ZFS mirror/RAID-Z），不做单盘；
 部署后若需并入现有集群当 OSS 节点，按第 10 章场景 A 执行（OST index 取现有最
大 +1）。
9.8 单机局限与替代方案
局限 说明 替代/规避
看不到横向扩展 聚合带宽=单机上限，无法演示"加 OST 涨
吞吐"
性能演示仍需多节点
MDS/OSS 资源竞争 元数据与数据 IO 抢同一 CPU/内存/磁盘 磁盘分开（MGT/MDT/OST 独立
盘）已是最低要求
无故障隔离 单点，无法做 HA/切换演练 需要 HA 演练则用多机
锁争用被掩盖 单客户端/单机场景锁竞争不明显 多客户端并发测试需多机
如果测试机配置较强（≥8 核 / 32 GB），两个进阶玩法：
6. 在测试机上起 3 台 VM 复刻 3 节点架构（按 5.3 配置）：与第 8 章环境完全一致，
还能演示跨节点网络路径；
7. 把测试机并入现有集群当 OSS 节点：如果目的是扩容量而非独立测试，直接按第 10 章
场景 A 走（注意 OST index 取现有最大 +1）。
单机 recovery 窗口提示：重启后 OST 进入 recovery（默认最长 5 分钟），期间 lfs
df 暂缺该 OST，等窗口结束自动恢复——正常现象。


<!-- 第 45 页 -->

45
第 10 章 扩容：新增 OSS 节点与数据盘
基于 2026-08-20 实测的 3 节点集群（第 8 章：mds1/oss1/client1，
192.168.12.220/221/222，Lustre 2.17.0 + ZFS 2.2.2）编写，覆盖两种扩容场
景。两种场景的本质相同——都是创建新的 OST。Lustre 的设计决定了扩容是"加目标"而
非"改配置"：新 OST 挂载后由 MGS 自动感知、容量自动并入，其他节点（含客户端）无
需任何改动。
10.1 扩容原理与两种路径
维度 说明
扩容原理 新增 OST（mkfs.lustre --ost）→ 挂载 → MGS 自动注册 → 容量与带宽自动并入
场景 A 耗时 全新节点约 1~1.5 小时（大头是服务端源码编译，约 40 分钟）
场景 B 耗时 现有节点加盘约 10 分钟
关键约束 新 OST 的 --index 必须为当前集群最大 index + 1，且永不复用已删除的 index
扩容后注意 新文件可能落在新 OST（默认布局由 MDS 按空间自动选），已有文件不会自动迁移，
需 lfs migrate（10.5）
10.2 扩容前置检查
```bash
# [mds1] 确认当前已注册目标与最大 OST index
sudo lctl get_param mgs.MGS.live.* # 期望列出 lustre-MDT0000 / OST0000 /
OST0001
# [client1] 确认当前容量基线（扩容后对比用）
lfs df -h /mnt/lustre
# [新节点 或 oss1] 确认新数据盘为裸盘（未分区、未格式化、未挂载）
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
sudo fdisk -l /dev/sdX # 按实际设备名核对
⚠ 若之前曾删除过 OST，lfs df -h 会显示 index 空洞，新 index 取"当前活跃最大
index + 1"即可（删除的 index 不复用）。
10.3 场景 A：新增 OSS 节点（oss2）
以新增 oss2（IP 192.168.12.223，带 2 块数据盘 /dev/sdb、/dev/sdc） 为例。
IP、用户名、盘符请按实际替换。流程与第 8.3 节在 oss1 上的执行完全一致，差异仅
在 index 起始值和主机名。
```


<!-- 第 46 页 -->

46
（1）环境探查与内核修正 [oss2]：同 9.4 步骤 1（探查 + 切 6.8 GA + purge HWE；
仅当内核为 7.0 HWE 时执行）。
（2）基础环境 [oss2]：
sudo hostnamectl set-hostname oss2
```bash
# /etc/hosts：三机映射 + 新节点自身（所有现有节点也要加 oss2 一行；删除默认 127.0.1.1 行）
sudo tee -a /etc/hosts <<'EOF'
192.168.12.220 mds1
192.168.12.221 oss1
192.168.12.222 client1
192.168.12.223 oss2
EOF
sudo sed -i '/^127.0.1.1 /d' /etc/hosts
# 时间同步 + 编译工具（同 9.4 步骤 2）
（3）ZFS 与编译依赖 [oss2]：同 9.4 步骤 3（注意 libzfslinux-dev module-assistant
debhelper quilt 实测必需）。
（4）服务端源码编译 [oss2]：同 9.4 步骤 4（走代理 clone，--with-zfs --disable-
ldiskfs，make debs -j4 约 40 分钟）。
（5）安装服务端并加载模块 [oss2]：同 9.4 步骤 5 前半（dpkg -i *.deb → depmod -a
→ modprobe → lctl version）。
（6）LNet 配置与连通验证 [oss2]：
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lctl list_nids # 期望 192.168.12.223@tcp
sudo lnetctl ping 192.168.12.220@tcp # 与 MGS(mds1) 互通，扩容前提
sudo lnetctl ping 192.168.12.221@tcp # 与现有 OSS 互通
（7）建 ZFS 池 [oss2]：
# 2 块数据盘 → 2 个独立池（单盘单池，与 oss1 保持一致）
sudo zpool create lustre-ost2 /dev/sdb
sudo zpool create lustre-ost3 /dev/sdc
sudo zpool list
生产环境建议用 mirror/RAID-Z 冗余而非单盘池。多盘合并建池也可，但单盘单池更利
于按 OST 粒度管理。
（8）创建并挂载 OST（index 从 2 开始）[oss2]：
```


<!-- 第 47 页 -->

47
```bash
# mkfs：--index 取当前最大 index(1) + 1，--mgsnode 指向 mds1
sudo mkfs.lustre --fsname=lustre --ost --index=2 \
--mgsnode=192.168.12.220@tcp --backfstype=zfs lustre-ost2/ost
sudo mkfs.lustre --fsname=lustre --ost --index=3 \
--mgsnode=192.168.12.220@tcp --backfstype=zfs lustre-ost3/ost
# 挂载
sudo mkdir -p /mnt/ost2 /mnt/ost3
sudo mount -t lustre lustre-ost2/ost /mnt/ost2
sudo mount -t lustre lustre-ost3/ost /mnt/ost3
mount | grep lustre
# 注册验证 [mds1]：此时 MGS 已自动感知新 OST
sudo lctl get_param mgs.MGS.live.* # 期望新增 lustre-OST0002 / lustre-OST0003
生产 HA 环境：mkfs 时建议加 --servicenode=<主 NID> --servicenode=<备 NID> 声明故
障切换伙伴。
（9）开机自启 [oss2]：fstab 追加两个 OST 挂载行 + zpool set cachefile=...
lustre-ost2 lustre-ost3 + lnet-setup.service（同 9.4 步骤 8，含
ExecStartPre=/usr/sbin/modprobe lnet）。
（10）扩容总验证：
# [client1] 容量：应新增 2 个 OST
lfs df -h /mnt/lustre # 期望出现 OST0002 / OST0003
# [client1] 新文件显式使用新 OST
sudo mkdir -p /mnt/lustre/test2
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test2
sudo dd if=/dev/zero of=/mnt/lustre/test2/newfile bs=1M count=512 oflag=direct
conv=fdatasync
lfs getstripe -v /mnt/lustre/test2/newfile # obdidx 应含 2 和 3
# 重启自愈抽验 [oss2]
sudo reboot
# 重启后：zpool 自动导入、OST 自动挂载
sudo lctl dl | grep -cE ' UP '
mount | grep lustre
10.4 场景 B：现有 oss1 加数据盘（快速路径）
不需要重新编译任何东西。核心三步：建池 → mkfs → 挂载。以新增一块盘 /dev/sdd
（新 OST index=2）为例。
# [oss1] 确认新盘
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE # 确认 /dev/sdd 为裸盘、未被挂载
```


<!-- 第 48 页 -->

48
```bash
# 建池、mkfs、挂载
sudo zpool create lustre-ost2 /dev/sdd
sudo mkfs.lustre --fsname=lustre --ost --index=2 \
--mgsnode=192.168.12.220@tcp --backfstype=zfs lustre-ost2/ost
sudo mkdir -p /mnt/ost2
sudo mount -t lustre lustre-ost2/ost /mnt/ost2
mount | grep lustre
# 自启与验证
# fstab 追加：lustre-ost2/ost /mnt/ost2 lustre defaults,_netdev 0 0
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-ost2
# [client1] 验证
lfs df -h /mnt/lustre # 期望新增 OST0002
⚠ 若场景 A 已把 index 用到 3，此处应取 lfs df -h 当前最大 index + 1（如
4），保证 index 全局唯一。
10.5 扩容后：新数据如何利用新 OST
新增 OST 后容量立即并入，但"文件落在哪个 OST"由布局（layout）决定：
新文件（默认行为）：目录/文件未显式设条带时，默认 stripe_count=1、stripe_offset=-
1，由 MDS 自动选择 OST（倾向剩余空间充足的）。新 OST 空间最大，新文件大概率自动
落在新 OST，但不保证——需要确定性分布时显式指定：
# [client1] 目录级默认布局：该目录下新文件全部用满所有可用 OST（含新增的）
sudo lfs setstripe -c -1 -S 4M /mnt/lustre/dataset
# 或固定用前 N 个 OST（按需）
sudo lfs setstripe -c 4 -S 4M /mnt/lustre/dataset
已有文件迁移（lfs migrate）：已有文件不会自动迁移到新 OST。要让老数据利用新 OST
的容量/带宽，需显式迁移（会重写数据，IO 期间性能下降，建议低峰执行）：
# 迁移单个文件：重新条带到 2 个 OST（MDS 自动选，通常含新 OST）
lfs migrate -c 2 /mnt/lustre/test/bigfile
# 迁移整个目录（注意：目录内大量小文件时锁争用明显）
lfs migrate -c -1 /mnt/lustre/test
# 迁移后验证
lfs getstripe -v /mnt/lustre/test/bigfile # obdidx 应含新 OST index
迁移前提：目标 OST 数量 ≤ 当前可用 OST 总数；lfs migrate 在线执行，文件可继续
被读取（写期间短暂不可写）。
```


<!-- 第 49 页 -->

49
第四篇 运维与排障
第 11 章 性能调优
11.1 条带参数调优
 大文件 / 高吞吐顺序 IO：增大 stripe_count（如 -1 用满所有 OST）与 stripe_size
（4~8 MiB），聚合多 OSS 带宽；
 小文件 / 海量元数据：保持 stripe_count=1，避免单个小文件跨多 OST 带来的额外开
销与空间浪费；
 目录级默认布局按工作负载分区设置（如 /dataset 设大条带，/home 设单条带）；
 善用 PFL 让文件随增长自动扩展条带宽度（见 2.3）。
11.2 LNet 与网络调优
 优先选用 o2ib/RoCE 等 RDMA 网络，显著降低延迟；
 配置多 NI 与 LNet Router 隔离管理/存储流量；
 调大 LNet 接收/发送缓冲区与 peer_credits、credits，提升并发 RPC 能力；
 2.17 支持动态 NID 配置，网络变更无需重建 target。
11.3 OSS/OST 数量与容量规划
 聚合带宽 ≈ OST 数 × 单 OST 带宽，规划时按吞吐目标反推 OST 数；
 单 OSS 管理 OST 数受内存与 CPU 约束，避免单点承载过多；
 后端文件系统参数：ldiskfs 调整 stride/stripe-width 对齐 RAID；ZFS 调整
recordsize、arc_max（控制 ARC 缓存占用，避免与 Lustre 缓存争内存）；
 磁盘调度器：HDD 用 mq-deadline，SSD/NVMe 用 none/noop。
11.4 客户端调优
```bash
# RPC 大小与每 RPC 页数的关系
lctl set_param osc.lustre-*.max_pages_per_rpc=256 # 影响 rpc_size（256 页*4K=1MB
RPC）
lctl set_param osc.lustre-*.max_rpcs_in_flight=8 # 并发在途 RPC 数
# 预读（readahead）提升顺序读
lctl set_param llite.lustre-*.max_read_ahead_mb=256
 max_pages_per_rpc 决定单 RPC 携带数据量（与底层网络 MTU/带宽匹配）；
 max_rpcs_in_flight 提升并发度，但消耗客户端内存；
 合理预读显著加速大文件顺序读取。
```


<!-- 第 50 页 -->

50
11.5 元数据性能与 DNE
 元数据瓶颈集中在 MDT；使用 DNE（分布式命名空间） 将不同目录子树分布于多个
MDT，提升并发；
 将热点目录（如临时目录）单独分配到专用 MDT；
 使用 SSD/NVMe 承载 MDT，降低随机小 IO 延迟；
 减少不必要的 stat 风暴与深度目录遍历（2.16+ 已优化大目录遍历）。
DNE 相关命令：
```bash
# 把某个大目录分布到 4 个 MDT（index 0,1,2,3）
lfs setdirstripe -D -c 4 /mnt/lustre/bigdir
# 2.15+ 支持按空间自动平衡新建目录到各 MDT
lfs setdirstripe -D --auto /mnt/lustre/work
11.6 关键监控指标
通过 lctl get_param 读取：
指标 获取方式 含义
OST/MDT 容量 lfs df -h 各目标使用率
IO 统计 lctl get_param osc.*.stats 读写次数/字节
RPC 情况 lctl get_param osc.*.rpc_stats（2.14+；早期版本用
osc.*.stats）
RPC 延迟分布
锁统计 lctl get_param ldlm.*.stats / ldlm_namespaces 锁授予/冲突
连接状态 lctl dl；lctl get_param *.*.ping 设备存活
客户端列表 lctl get_param mdt.*.exports 已连接客户端
基准工具：IOR（并行带宽）、mdtest（元数据）、fio、dd。示例：ior -w -r -t 1m -b
1g -o /mnt/lustre/test.dat。监控对接：Prometheus + Grafana（采集 osc.*.stats 与
mdt.*.md_stats）。
```


<!-- 第 51 页 -->

51
第 12 章 日常运维
12.1 集群状态检查
lctl dl # 本节点设备清单及状态（UP/CLEAN 为正常）
lfs df -h /mnt/lustre # 各 OST 容量
lctl get_param mdt.*.exports # 查看连入的客户端
cat /proc/fs/lustre/health_status # 节点健康状态
关注 lctl dl 中设备是否 UP；lfs df 是否有 OST 显示 0 或异常。
12.2 增加与移除 OST
新增 OST（扩容）：见第 10 章——格式化新盘并挂载，MGS 自动感知，无需停机。
移除 OST（缩容）：必须先迁移其上数据，避免文件条带丢失：
```bash
# 将目录下文件迁移到新的 4 个 OST（示例）
lfs migrate -n 4 /mnt/lustre/dataset
# 或使用 Robinhood 策略引擎自动迁移
# 置为 degraded/offline 后卸载（生产务必先排空数据）
⚠ 数据迁移期间目标目录 IO 性能会下降，建议在业务低峰期执行；迁移完成后通过
lfs getstripe 验证布局变更。
12.3 日志查看与故障排查思路
 内核日志：dmesg、/var/log/messages、journalctl -k；Lustre 专属日志通常以
Lustre:/LustreError: 前缀；
 常见错误：
 LustreError: ... evicted：客户端被驱逐，检查网络/LNet 与 recovery 超时；
 connection to <nid>@... was lost：LNet 连通性故障；
 no route to <nid>：LNet 路由缺失或 NI 未配置；
 OOM：多为 max_pages_per_rpc / ZFS ARC 占用过高，需下调。
典型排查流程：
 客户端挂载失败：① lctl ping <mgsnid> 确认 MGS 可达 → ② lctl network up 确认
LNet → ③ lctl dl 检查 target 状态 → ④ 查看日志具体报错；
 LNet 连接问题：lctl list_nids 确认本端 NID；lnetctl ping <peer_nid> 测试对
端；核对 networks= 配置与路由表；
 性能骤降：检查是否热点 OST（某些 OST 接近满）、锁争用（ldlm 统计）、网络丢
包。
```


<!-- 第 52 页 -->

52
12.4 备份与恢复
 元数据备份：定期对 MDT 做 ZFS 快照 / dump；lfs 布局信息包含在元数据中；
 配置导出：lctl conf_param 配置存于 MGS，需备份 MGT；可记录 mkfs.lustre 参数以
便重建；
 数据备份：OST 数据通常按文件级策略备份（结合 Robinhood 策略引擎做分层/归
档），而非逐盘镜像；
 恢复演练：保留 mkfs.lustre 原始命令与 index，重建 target 后由 MGS 重新下发配
置。
12.5 参数持久化
 临时参数：lctl set_param（仅当前运行有效，重启丢失）；
 持久化参数：lctl conf_param <target>.<subsystem>.<param>=<value> 写入 MGS 配置
日志，由 MGS 下发到对应服务，重启后依然生效；
 全局服务配置也可写入 /etc/modprobe.d/lustre.conf（发行版相关）。


<!-- 第 53 页 -->

53
第 13 章 故障排查速查总表
排障总原则：dmesg | tail、journalctl -xe、/var/log/messages 三个日志先行；其次
lctl dl 看目标状态；最后检查网络（lnetctl ping、nc -vz <ip> 988）。
13.1 模块与版本问题
现象 原因 排查 解决
modprobe
lustre 报
Invalid module
format
内核头文件与运行
内核不一致，或模
块针对不同内核编
译
uname -r；rpm -q
kernel-devel / apt
list --installed \
grep linux-headers 重装与
uname -r 一致的 kernel-
devel/headers 后重编重装；服务
端必须用 Whamcloud 补丁内核
（RHEL）或 GA 内核（Ubuntu）
Unknown symbol
/ no symbol
version
模块依赖缺失或顺
序错误
按序手动 modprobe
libcfs; modprobe
lnet; modprobe
lustre
depmod -a 重建依赖；lsmod 检查
旧模块
找不到
e2fsprogs 特有
版本 / mkfs 报
e2fsprogs too
old
没装 Whamcloud
定制 e2fsprogs
e2fsprogs -V（检查
wc 后缀）
RHEL：启用 e2fsprogs-wc 仓库；
Ubuntu：按 7.3 路线 B 编译安装
模块加载了但
lctl version 报
错
用户态工具与内核
模块版本不匹配
rpm -q lustre kmod-
lustre
统一升/降到同一版本号
modprobe
lustre 报
Required key
not available
/ module
verification
failed
Secure Boot 拒
绝未签名模块
mokutil --sb-state 实验环境在 UEFI 固件中关闭
Secure Boot；或注册 MOK
Module lustre
not found
deb/rpm 未安
装，或 depmod
未运行
find
/lib/modules/$(uname
-r) -name '*lustre*'
执行 dpkg -i debs/*.deb（或
make install），然后 sudo
depmod -a
13.2 编译与依赖问题（Ubuntu 源码编译）
现象 原因 解决
configure --with-zfs 报
missing zfs development
headers
缺 ZFS 用户态开发头文件 sudo apt install libzfslinux-
dev；仍找不到则 --with-
zfs=/usr/src/zfs-<版本> 显式指定
make debs 报 Unmet build
dependencies
缺 module-assistant /
debhelper / quilt
sudo apt install module-assistant
debhelper quilt（必要时补 dpkg-
dev）
ldiskfs 编译失败：找不到
fs/ext4 或 patch FAILED
未做 7.3 路线 B 的 ext4
链接；或 linux-source 小
版本差异过大
完成链接；换与 uname -r 一致的小版
本源码包重试
make debs 报缺 dpkg-
buildpackage / dpatch
未装打包工具 sudo apt install -y dpkg-dev（必
要时 dpatch）
gcc 相关编译错误（struct
module has no member）
编译器版本与内核工具链不匹
配
sudo update-alternatives --config
gcc 切到 gcc-13，make clean 后重编
apt install lustre-
client-utils 报
旧系统装 noble 仓库包 换对应发行版的仓库或用源码编译
（24.04 自带 glibc 2.39 满足）


<!-- 第 54 页 -->

54
GLIBC_2.38 not found
GitHub git clone 极慢/卡
住
直连被限速 走代理 export
https_proxy=http://<proxy>:<port>
13.3 内核与平台问题（Ubuntu）
现象 原因 解决
HWE 内核下 DKMS 编译失败或模
块加载失败
模块仓库只覆盖 GA 6.8 线 切回 GA 内核：装 linux-
image-generic/linux-
headers-generic，移除 HWE
元包，GRUB 选 6.8 启动后
apt-mark hold
zfs-dkms 安装卡死、dpkg 锁被
占
在 7.0 HWE 内核上构建 ZFS 失
败（仅支持内核 ≤6.6）
先切 6.8 GA 并 purge 7.0 再
装 ZFS；卡死时 pkill -f
"dkms build -m zfs" + dpkg
--configure -a
GRUB 切换后仍进 7.0 菜单项版本号与 ls
/boot/vmlinuz-* 实际不符
用实际版本号重写
GRUB_DEFAULT，update-grub
后重启
编译 OOM / IO 卡顿 VM 内存不足（4 GB 偏紧） 调高内存至 8 GB；关闭 GUI/后
台程序；确认 swap 已关
dd 首次写满突然变慢 thin 虚拟盘首次分配块 介意则换 thick 盘
整机快照回滚后无法挂载 MGS 配置日志与目标代次失配 不要对运行中的 Lustre VM 做
快照/回滚；只有整机重做
target 或 mkfs.lustre --
writeconf（谨慎）
13.4 时间同步与客户端 evict
现象 原因 排查 解决
客户端频繁报
LustreError: ...
timed out、
recovering、evicted
节点时钟偏差超过恢复
窗口，服务端判定客户
端"失联"
timedatectl；
chronyc sources -v
全集群 chrony 对齐同
一时间源；被 evict
后 umount 再 mount
恢复
挂载后 30~90 秒无操
作即报超时
NTP 服务未启动或指向
不同源
systemctl status
chrony(d)
修正配置后重启
chrony
所有节点时间稳定但客
户端仍周期性 evict
时钟漂移叠加 VM 暂停/
迁移
chronyc tracking 看
offset
校正时间源；避免 VM
挂起（suspend）
13.5 LNet 与网络问题
现象 原因 排查 解决
lctl list_nids 为空 LNet 未配置/未 up lnetctl net show lnetctl lnet
configure 后
lnetctl net add --
net tcp0 --if
<ifname>
lnetctl ping 不通 NID 拼写/网卡名/网段
错误
ip a；lnetctl net
show --detail
核对网卡名与 IP；--
if 填实际接口
ping 通但 mount 失败 防火墙挡 988/tcp nc -vz <ip> 988 RHEL：firewall-cmd
--permanent --add-
port=988/tcp；


<!-- 第 55 页 -->

55
Ubuntu：sudo ufw
allow 988/tcp
Connection timed
out 偶发
网卡 offload 导致大
包丢包（VMware 常
见）
ethtool -k <if> ethtool -K <if> tx
off rx off 验证
NID 写成 @tcp1 而实
际是 @tcp0/@tcp
LNet 网络名不一致 lctl list_nids 对比
两端
全集群统一网络名
（2.17 默认显示
@tcp）
VirtualBox 下 ping
不通
网络模式非桥接 ping <ip>；ip route 改用 Bridged，静态
IP
13.6 挂载与目标问题
现象 原因 排查 解决
mount.lustre:
Connection timed
out
MGS 不可达/未启
动/防火墙
lnetctl ping <mgs-
nid>；nc -vz <mgs-ip>
988
启动 MGS（挂载 MGT）；放行端
口
mount.lustre:
not a Lustre
filesystem
设备不是 Lustre
target，或磁盘
已被破坏
mkfs.lustre --dryrun
/dev/sdX
用 mkfs.lustre 重新格式化
（注意数据）
mount.lustre:
Can't contact
MGS
服务端 LNet 未
起来，或 --
mgsnode 写错
在 MGS 上 lctl
list_nids
修正 MGS NID 后重新挂载
mount.lustre:
mount option
error
fstab 选项写错 grep lustre /etc/fstab 改回 defaults,_netdev
重启后客户端挂载
丢失 / EBUSY
（rc=-16）
MGS recovery
窗口期内客户端提
前挂载；fstab
_netdev 不等服
务
lctl get_param
mdt.lustre-
MDT0000.recovery_status
确认 COMPLETE
用"先 ping MGS 直到可达再
mount"的等待式 systemd 服务
（见 8.5）
OST 状态为 IN
（inactive）或缺
失
OST 未挂载、或
MGS 尚未下发配
置
lctl dl；在 OSS 上挂载该
OST
挂载该 OST；等待数秒后复查
新增 OST 后 lfs
df 不显示
客户端缓存了旧配
置
lctl dl 确认 osc 存在 umount/mount 客户端；或核对
MGS 记录
mkfs 报 index 已
占用 / 已存在
索引冲突（不同
OST 同 index）
或复用了已删除
index
lctl get_param
mgs.MGS.live.*
重新指定未占用 index（当前最
大 +1）
OST 挂载后随即
LustreError: ...
expired
生成代次
（generation）
与 MGS 记录不符
（如做过快照回
滚）
服务端日志找 generation
字样
mkfs.lustre --writeconf 重
置（谨慎，会丢弃该 target 的
旧布局配置）
重启后 fstab 里
/dev/sdb 变成了
别盘
VM 磁盘枚举顺序
在重启后变化
lsblk -f；ls -l
/dev/disk/by-id/
用 UUID= 或 /dev/disk/by-
id/... 替换 fstab 与建池路
径；建池前重新 lsblk 确认映射
lfs df 只显示部
分 OST
有 OST 未挂载 mount \ grep lustre、lctl dl 核对目
标状态，挂载缺失的 OST
单大盘分区部署
zpool create 报
设备忙
分区被系统占用/
挂载
lsblk 确认分区未被挂载 卸载后重试


<!-- 第 56 页 -->

56
性能忽高忽低 磁盘稀疏文件扩
展、内存气球、宿
主机争抢
iostat -x 1；free -h 换 thick 盘、给足内存、错峰实
验


<!-- 第 57 页 -->

57
附录
附录 A 部署前检查清单
部署前
 ☐ 已确认发行版 + Lustre 版本 + 内核版本三者匹配（第 4 章）
 ☐ 已登录 downloads.whamcloud.com 核对仓库中确切的 kernel/包版本号（RHEL
系）
 ☐ Ubuntu：已确认使用 GA 6.8 内核，且先切内核、purge HWE，再装 ZFS
 ☐ 每台节点静态 IP、/etc/hosts 全集群一致
 ☐ 所有节点 chrony 已指向同一时间源并 chronyc sources 显示同步
 ☐ SELinux 已 disabled（RHEL 系）；firewalld/ufw 已关闭或放行 988/tcp
 ☐ MGT/MDT/OST 各用独立虚拟盘/独立盘，已重新 lsblk 确认设备名映射并记录
 ☐ 明确告知团队：不做 VM 整机快照回滚
部署中（按序）
 ☐ 内核就位（RHEL：补丁内核 + grubby --set-default + dnf.conf 锁定；Ubuntu：
GA 内核 + hold + purge HWE）
 ☐ Whamcloud e2fsprogs 已安装（RHEL 系 / Ubuntu ldiskfs 路线）
 ☐ ZFS 或 ldiskfs 后端依赖已装齐（Ubuntu：libzfslinux-dev module-assistant
debhelper quilt）
 ☐ modprobe lustre 无报错，lctl version 版本正确
 ☐ LNet 已 up，lctl list_nids 输出正确 NID，lnetctl ping 全互达
 ☐ MGS/MGT 挂载成功；MDT(index 0) 挂载成功；OST(index 0,1,...) 挂载成功
 ☐ mds1 上 lctl dl 显示 MGS/MDT/OST 全部 UP；lctl get_param mgs.MGS.live.*
注册齐全
 ☐ 客户端 mount -t lustre <mgs>:/<fs> /mnt/lustre 成功
 ☐ lfs df -h 显示 MDT 与全部 OST
 ☐ lfs setstripe -c 2 + dd 读写通过，lfs getstripe -v 确认条带分布
 ☐ fstab 已写入（服务端目标 + ZFS cachefile）；客户端挂载用等待式 systemd
服务
部署后
 ☐ 执行一次整机重启（含多台同时重启场景），确认 LNet、所有目标、客户端挂载自动
恢复
 ☐ 备份方式已改为 Lustre 原生手段（非 VM 快照）


<!-- 第 58 页 -->

58
附录 B 实测经验教训汇总
以下教训来自 2026-08-20 三节点集群与 2026-08-26 单机（192.168.12.160）两次实
测：
8. 磁盘设备名漂移：内核切换/重启后 /dev/sdX 可能互换（实测 sdb↔sdc、sdd↔sde），
建池前必须重新 lsblk 确认；生产用 /dev/disk/by-id/ 建池。池创建后 vdev 绑定
设备内部 ID，后续重启不受影响。
9. 内核切换顺序：必须先切 6.8 GA 并 purge HWE 7.0，再装 ZFS；顺序反了 zfs-dkms
编译卡死（dpkg 锁被占）。修复：pkill -f "dkms build -m zfs" + dpkg --configure
-a。
10.客户端挂载方式：fstab _netdev 不等 MGS recovery，重启（尤其多台同时重启）会
丢挂载；用"先 ping MGS 直到可达再 mount"的等待式 systemd 服务。
11.编译依赖易缺：Ubuntu 上 libzfslinux-dev（configure 报 ZFS headers 缺失）、
module-assistant debhelper quilt（make debs 报 Unmet）实测必需；GitHub clone
走代理。
12.sudo 注意：普通用户 modprobe lustre 报 Operation not permitted，需 sudo；sudo
下 ~ 变 /root，涉及 ~ 的命令用绝对路径；systemctl reboot 比 reboot 可靠。
13.NID 网络名：方案示例写 @tcp0，Lustre 2.17 实际显示 @tcp，二者对应同一网络，互
ping/挂载统一用实际显示值。
14.MDT 偏小可重建：MDT 只有 2G 时，销毁四池按正确盘符重建即可（实验环境数据量
小，10 分钟完成）。
15.recovery 窗口是正常现象：重启后 MDT/OST 进入 recovery（默认最长 5 分钟），
期间客户端连接被拒（EBUSY）、lfs df 暂缺该 OST，等窗口结束自动恢复。
16.lnet-setup.service 重启后 failed：LNet 模块随挂载已加载时不影响功能；如需严
格自启，在 unit 中追加 ExecStartPre=/usr/sbin/modprobe lnet。


<!-- 第 59 页 -->

59
附录 C 术语与常用命令速查
C.1 组件术语
术语 全称 说明
MGS / MGT Management Server / Target 集群配置管理服务/目标，存放 config log，挂载
入口
MDS / MDT Metadata Server / Target 元数据服务/目标，管命名空间、inode、布局
OSS / OST Object Storage Server /
Target
对象存储服务/目标，真正存数据
DNE Distributed Namespace 分布式命名空间，多 MDT 分片目录树
PFL Progressive File Layout 渐进式文件布局，按文件大小分段条带
FLR File Level Redundancy 文件级冗余（lfs mirror）
LNet Lustre Networking 网络抽象层（tcp/o2ib/RoCE）
NID Network Identifier 节点网络标识（IP@网络类型）
LDLM Lustre Distributed Lock
Manager
分布式锁管理器
FID / OID File / Object Identifier 文件/对象唯一标识
C.2 mkfs.lustre 关键参数
参数 作用
--fsname 文件系统名（如 lustre），MGS 据此区分多个文件系统
--mgs 标记该目标为 MGT（管理目标）
--mdt / --ost 标记为元数据目标 / 对象存储目标
--index 目标唯一索引（MDT/OST 各自编号，必须唯一且稳定）
--mgsnode MGS 的 NID（OST/MDT 据此找到 MGS）
--servicenode / --
failnode
故障切换伙伴 NID（生产环境建议配置）
--backfstype=zfs 指定后端为 ZFS（默认 ldiskfs）
--mkfsoptions 透传给底层后端文件系统的格式化选项
--writeconf 重置 target 配置代次（谨慎使用）
C.3 日常管理命令
```bash
# lctl：底层设备与参数
lctl dl # 列出本节点所有 Lustre 设备（target/客户端）
lctl get_param *.*.stats # 读取各类统计
lctl set_param osc.*.max_pages_per_rpc=256 # 动态调参（重启失效）
lctl conf_param lustre.ost.ost_io.nrs_crr_n='' # 写入 MGS 配置（持久化下发）
lctl network up # LNet 启动
lctl list_nids # 列出本端 NID
lctl ping <nid> # 连通性测试
# lfs：文件级与布局管理
lfs df -h /mnt/lustre # 查看各 OST 容量与使用情况
lfs getstripe [-v] <path> # 查看布局
lfs setstripe -c N -S SIZE <path> # 设置条带
lfs setdirstripe -D -c N <dir> # 大目录分布到多 MDT（DNE）
lfs migrate -c N <path> # 迁移文件到 N 个 OST
```


<!-- 第 60 页 -->

60
lfs quota -u username /mnt/lustre # 配额查询
```bash
# lnetctl：LNet 动态配置
lnetctl lnet configure # 初始化 LNet
lnetctl net add --net tcp0 --if <网卡> # 添加网络
lnetctl net show # 查看网络状态
lnetctl ping <nid> # 连通性测试（2.13+）
# zfs/zpool：ZFS 池管理（ZFS 后端）
zpool create <pool> <device> # 建池
zpool list # 查看池
zpool set cachefile=/etc/zfs/zpool.cache <pool> # 写入缓存文件（开机自动导入）
```


<!-- 第 61 页 -->

61
附录 D 参考资料
资料 地址 用途
Lustre
官网
https://www.lustre.org/ Releases Roadmap、
Download、Documentation
入口
Lustre
Wiki（社
区主仓
库）
https://wiki.lustre.org/ Architecture、
Administration、LNET、
Monitoring 等专题
Whamclou
d 下载站
https://downloads.whamcloud.com/ 所有预编译包/SRPM
Whamclou
d 支持矩
阵
https://wiki.whamcloud.com/display/PUB/Lustre+Sup
port+Matrix
版本×发行版×内核
×ZFS/e2fsprogs 权威对照
Whamclou
d Wiki /
JIRA
https://wiki.whamcloud.com/ 、
https://jira.whamcloud.com/
问题跟踪与开发任务（LU-
18010：Ubuntu 构建）
LU-18010 https://jira.whamcloud.com/browse/LU-18010 Ubuntu 24.04 服务端构建：
ext4 链接法、gcc-13
OpenSFS https://www.opensfs.org/ 社区协调组织
Lustre
源码仓库
https://github.com/lustre/lustre-release.git 服务端/客户端源码（分支
b2_16 / b2_17 / master）
e2fsprog
s 源码
https://review.whamcloud.com/tools/e2fsprogs（分支
master-lustre）
Whamcloud 定制 e2fsprogs
Azure
Managed
Lustre
客户端仓
库
https://packages.microsoft.com/repos/amlfs-
\<codename\>/
Ubuntu 客户端 DKMS 包
（Microsoft Learn 检索文
档）
Google
Cloud
Lustre
文档
https://cloud.google.com/lustre/docs 客户端 DKMS 仓库（lustre-
client-ubuntu-noble）
Ceph 官方
文档
https://docs.ceph.com/ 第 3 章选型对比 Ceph 侧参
考
Robinhoo
d Policy
Engine
https://github.com/cea-hpc/robinhood 大规模 Lustre 数据管理与分
层
Lustre
系统管理
指南
wiki.lustre.org 内 Lustre Systems Administration
Guide
生产部署实践
Lustre
File
System
Internal
s（第二
版）
书籍 理解 LDLM、布局、恢复的内
部机制
中国
Lustre
用户组
https://www.lustrefs.cn/ 中文社区
*本手册技术基线：Lustre 2.15.8 LTS（RHEL 系）/ 2.17.0（Ubuntu 系）、ZFS
2.2.2、Ubuntu 24.04 / Rocky 8.10。整理日期：2026-09-18。*
