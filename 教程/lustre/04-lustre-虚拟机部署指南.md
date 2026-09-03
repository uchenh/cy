# Lustre 分布式文件系统虚拟机部署指南

> 本文面向**有 Linux 基础、希望在虚拟机（KVM / VMware / VirtualBox）环境从零搭建一套可用 Lustre 集群**的工程师，目标是让你用 1~4 台 VM 跑起一个可用于学习、实验与功能验证的最小集群（含 MDS、OSS、客户端挂载与基本 IO 测试）。
>
> 全文按「先选版本 → 再定虚拟化配置 → 然后按拓扑部署 → 最后排障」的顺序组织。每一步都给出可复制执行的命令；括号内标注「单节点」与「多节点」的差异。
>
> 📚 **配套资料**：本文档是 Lustre 文档集的**实操部署篇**。建议搭配阅读：
> - [01-lustre-知识点详解.md](./01-lustre-知识点详解.md) — Lustre 架构与运维详解
> - [02-lustre-研究规划.md](./02-lustre-研究规划.md) — 学习路线
> - [03-lustre-vs-ceph-对比.md](./03-lustre-vs-ceph-对比.md) — 与 Ceph 的选型对比
> - 工作区 `docs/Lustre_Manual_cn_0.0.3.pdf` — 官方手册中文版（概念、`mkfs.lustre` 参数、`lctl` 命令的权威参考）
>
> ⚠️ **免责声明**：本文给出的是官方/社区已验证的通用路径。具体软件包的**确切小版本与内核版本**会随时间变化，动手前请以 [Whamcloud 下载站](https://downloads.whamcloud.com) 与 [Lustre 官方支持矩阵](https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix) 为准；文中命令中的版本号（如 `2.15.8`、`4.18.0-553.82.1.el8_lustre`）系撰写时的最新 LTS 示例。

---

## 1. 文档范围与前置条件

### 1.1 本指南覆盖什么

- 在 VM 中部署由 **MGS（+MGT）/ MDS（+MDT）/ OSS（+OST）/ Client** 组成的最小 Lustre 集群；
- 完成**客户端挂载、条带化布局查看、`dd`/`fio` 基础 IO 验证**；
- 覆盖版本选择、内核模块匹配、LNet 网络配置、开机自启等**最容易踩坑**的细节。

### 1.2 前置知识与环境要求

| 项目 | 要求 |
| --- | --- |
| 前置知识 | Linux 基本操作（分区、mount、fstab、yum/dnf、systemd）；能看 `dmesg` 排障；了解内核模块概念 |
| 虚拟机管理程序 | KVM/QEMU、VMware Workstation/Fusion/ESXi、VirtualBox 任一即可；实验建议 KVM（性能与直通能力最好） |
| 实验机配置 | 宿主机 ≥ 8GB 内存、≥ 4 vCPU；所有 VM 合计磁盘 ≥ 60GB |
| 系统 ISO | 按 §2 版本矩阵选择：CentOS 7.9（旧）、Rocky/Alma/RHEL 8.10（推荐） |
| 网络 | 1 个虚拟交换机/网桥即可，所有 VM 同网段、静态 IP |
| 时间预算 | 首次部署（含系统安装）约 1~2 天；熟练后约 2~3 小时 |

### 1.3 实验拓扑规模建议

| 规模 | VM 数量 | 用途 |
| --- | --- | --- |
| 最小可跑 | 1 台（单节点，§4.1） | 验证 Lustre 基本功能、学习命令 |
| 推荐 | 3 台（MGS+MDS、OSS、Client） | 模拟真实部署，可演示扩展与故障 |
| 进阶 | 4 台（MGS+MDS、OSS×2、Client×1） | 验证多 OST 条带化、OST 故障影响 |

> 💡 若宿主机资源紧张，可以先做 **1 台单节点**打通流程，再平滑迁移到多节点（命令差异见 §4.3，几乎没有额外学习成本）。

---

## 2. 版本与兼容性（先讲清楚再动手）

### 2.1 推荐版本组合

Lustre 的服务端代码与 Linux 内核深度耦合，**服务端必须使用 Whamcloud 打过补丁的定制内核**（或自行编译内核模块）。因此「系统发行版 + Lustre 版本 + 内核版本」三者必须绑定。以下组合来自 [Whamcloud 官方支持矩阵](https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix)（RHEL 家族包含 CentOS/Rocky/Alma 衍生版）：

| 方案 | 发行版 | Lustre 版本 | 服务端内核（示例） | e2fsprogs | 结论 |
| --- | --- | --- | --- | --- | --- |
| **A（推荐）** | Rocky/Alma/RHEL **8.10** | **2.15.8**（当前 LTS，2025-12 发布） | `4.18.0-553.82.1.el8_lustre` | `1.47.3.wc2+`（走 Whamcloud `latest` 源即可） | 活跃维护、文档多、内核较新，**首选** |
| B（旧环境） | CentOS **7.9** | **2.12.9**（前代 LTS，2022-06 发布） | `3.10.0-1160.49.1.el7_lustre` | `1.46.2.wc5` | 大量旧教程基于此；EOL 后无安全更新，仅学习 |
| C（新内核） | RHEL/Rocky **9.x** | 2.16.x / 2.17.x（非 LTS 大版本） | `5.14.0-<build>.el9_lustre` | `1.47.3.wc2+` | 服务端上 EL9 需要 2.16+，社区优先推荐 LTS（2.15） |
| 客户端变体 | 客户端可放宽 | 2.15.x | 客户端可用**发行版原装内核**（patchless 客户端） | — | 2.15 客户端支持 EL8/EL9、SLES15 SP6、Ubuntu 22.04 |

> 📌 **版本对应关系要点**
> - **服务端三个角色（MGS/MDS/OSS）必须使用完全相同的 Lustre 版本与内核版本**，否则 `lctl dl` 会出现版本不匹配、目标挂载失败；
> - **客户端与服务端建议同版本**（官方允许客户端不高于服务端一个版本线内，但实验环境请保持完全一致，避免不必要排障）；
> - **内核与内核模块严格一一对应**：`uname -r` 必须等于 kernel-devel/kernel-headers 版本，也必须等于 `kmod-lustre*` 编译所针对的内核版本。系统更新内核后，Lustre 模块将无法加载——所以安装后要**锁定内核**（见 §5.2 第 3 步）。

### 2.2 Whamcloud 预编译包（推荐路线）

Whamcloud 为各发行版提供预编译仓库，路径规律：

```
https://downloads.whamcloud.com/public/lustre/lustre-<版本>/<发行版标识>/{server|client|patchless-ldiskfs-server}/
https://downloads.whamcloud.com/public/e2fsprogs/{latest|<版本号>}/<el版本>/
```

例如（撰写时的实际存在路径）：
- 方案 A：`https://downloads.whamcloud.com/public/lustre/lustre-2.15.8/el8.10/server/`、`.../el8.10/client/`；e2fsprogs 用 `https://downloads.whamcloud.com/public/e2fsprogs/latest/el8/`
- 方案 B：`https://downloads.whamcloud.com/public/lustre/lustre-2.12.9/el7.9.2009/server/`、`.../el7.9.2009/client/`；e2fsprogs 用 `https://downloads.whamcloud.com/public/e2fsprogs/1.46.2.wc5/el7/`

服务端常用包：`lustre`（元包）、`kmod-lustre`、`kmod-lustre-osd-ldiskfs`、`lustre-osd-ldiskfs-mount`、`lustre-resource-agents`（HA 用，非必需）；客户端：`lustre-client`（必要时 `lustre-client-dkms`）。另外**必须**装 Whamcloud 定制的 `e2fsprogs`（否则 `mkfs.lustre` 格式化 ldiskfs 会失败）。

> ⚠️ **安装前先核对仓库实际内容**：登录 `downloads.whamcloud.com` 对应目录，确认内核包名与版本号后再执行安装，不要照抄旧文档的包名。

### 2.3 什么时候需要自己编译内核模块

当**目标发行版/内核在 Whamcloud 仓库没有预编译服务端包**时（例如 Ubuntu 服务端、较新的 RHEL 小版本、非标准内核），需要从源码编译。编译要点（概要，非完整日志）：

1. **准备工具链与依赖**：`gcc gcc-c++ make libtool autoconf kernel-devel-$(uname -r)`，并安装 Whamcloud 定制 e2fsprogs（ldiskfs 后端依赖其特有 feature）；
2. **获取源码**：从 downloads.whamcloud.com 的 `SRPMS` 子目录下载与发行版匹配的 `lustre-<ver>.src.rpm`（2.15 起 ldiskfs 由 `lustre-ldiskfs-dkms` 提供）；
3. **配置**：指定与当前内核匹配的构建头文件路径：
   ```bash
   ./configure --with-linux=/usr/src/kernels/$(uname -r)   # 指向 kernel-devel 安装的内核头文件
   ```
4. **编译安装**：
   ```bash
   make && make modules            # 编译内核模块
   make modules_install            # 安装到 /lib/modules/<uname -r>/
   make install                    # 安装用户态工具（mkfs.lustre、lctl、lfs 等）
   depmod -a                       # 重建模块依赖，否则 modprobe 找不到新模块
   modprobe lustre && lsmod | grep lustre   # 验证加载
   ```
5. **验证**：`lctl version` 输出版本号即为成功。若 `modprobe` 报 `Invalid module format`，多为内核头文件与运行内核版本不一致，重新 `yum/dnf install kernel-devel-$(uname -r)` 后重编。

> ⚠️ 源码编译工作量不小（首次通常要半天到一天），**实验环境优先用预编译包**；只有目标发行版实在没有对应包时才编译。

---

## 3. 虚拟机配置建议

### 3.1 资源建议表

| 资源 | 最小可跑（1 台单节点） | 推荐（3 台多节点） |
| --- | --- | --- |
| CPU（每 VM） | 2 vCPU | MDS 4 vCPU / OSS 4 vCPU / Client 2 vCPU |
| 内存（每 VM） | 4GB | MDS 8GB / OSS 8GB / Client 4GB |
| 系统盘 | 20GB（thin） | 20GB（thin） |
| 数据盘 | MGT 1GB×1、MDT 10GB×1、OST 20GB×2 | 按角色：MDS 1+10GB；OSS 20GB×2~4 |
| 网卡 | 1 块 virtio/PVSCSI 等虚拟网卡 | 1 块（管理+数据共用）或 2 块（管理/数据分离，进阶） |

### 3.2 各角色配置建议

| 角色 | CPU | 内存 | 磁盘 | 说明 |
| --- | --- | --- | --- | --- |
| **MGS/MDS**（元数据） | 2~4 vCPU | 4~8GB | 系统盘 + **MGT（≥1GB，独立盘）** + **MDT（≥10GB，独立盘）** | 元数据操作（create/lookup）吃 CPU 与锁；内存用于 inode 缓存，给足 |
| **OSS**（数据） | 2~4 vCPU | 6~8GB | 系统盘 + **每块 OST 一块独立虚拟盘**（≥20GB，2~4 块） | 内存主要用做页缓存吸收写回；多块独立盘 = 多 OST = 体现条带化价值 |
| **Client** | 2 vCPU | 4GB | 系统盘 + 挂载点即可 | 客户端无需数据盘，只跑内核模块与 `lfs`/`dd`/`fio` |

> 💡 角色可以合并：MGS 与 MDS 常放同一节点（本文示例即如此）；资源紧张时 MGS/MDS/OSS 全合一（§4.1 单节点）。

### 3.3 磁盘规划

- **控制器选择**（影响性能与热插拔支持）：
  | 平台 | 推荐控制器 | 说明 |
  | --- | --- | --- |
  | KVM | **virtio-scsi** | 多队列、性能好；系统盘也可用 virtio-blk |
  | VMware | **PVSCSI** | 比默认 LSI 好；Workstation 亦可用 |
  | VirtualBox | **SATA（AHCI）或 NVMe** | 默认 SATA 即可，NVMe 性能略好 |
- **每类 target 一块独立虚拟盘**：MGT / MDT / 每个 OST 分别对应独立块设备（如 `/dev/sdb`=MGT、`/dev/sdc`=MDT、`/dev/sdd`=`/dev/sde`=OST）。**不要在系统盘上为多个 target 建 LVM 池或把盘做 RAID**，VM 里保持「一 target 一块盘」最直观；
- **LVM/分区**：实验环境建议**直接对整个块设备 `mkfs.lustre`**（最省事）。若想分区，务必每个 target 独占一个分区并记录清楚 `UUID`；LVM 逻辑卷也可用（需要时再扩展），但增加了排查复杂度；
- **稀疏文件 / 快照**：
  - 虚拟磁盘用 **thin/thick 预分配**均可；用 qcow2 稀疏文件能省宿主机空间，但**首次写满时会突然变慢**，介意就选 raw/thick；
  - ⚠️ **绝对不要对运行中的 Lustre 节点做整机快照/回滚**。Lustre 目标（MGT/MDT/OST）与 MGS 上记录的配置日志（`CONFIGS/*`）必须严格一致，回滚会让「目标磁盘内容」与「MGS 记录的生成代次（generation）」失配，导致整机无法挂载或数据损坏。**备份请用官方 `mkfs.lustre --writeconf` + 目标级 `lctl`/文件系统级工具**（如 `rsync` 客户端文件），而非 VM 快照。

### 3.4 网络

- **带宽**：实验 1GbE 足够（顺序读写在单 VM 下也能到数百 MB/s）；追求性能用 10GbE 或直通。
- **RDMA / o2ib 取舍**：

| 网络类型 | NID 示例 | VM 里是否可用 | 说明 |
| --- | --- | --- | --- |
| `tcp`（socklnd，默认端口 **988**） | `192.168.10.11@tcp` | ✅ 直接可用 | 实验首选，配置最简单 |
| `o2ib`（ko2iblnd，InfiniBand/RoCE） | `192.168.10.11@o2ib` | ⚠️ 需 SR-IOV 或 PCI 直通 HCA | 虚拟化环境极少用；没有真 IB 卡/直通就别碰 |

  结论：**实验环境一律用 `@tcp`**。VM 里做 o2ib 需要把 IB HCA 通过 PCI 直通/SR-IOV 透传给 VM，还依赖宿主机驱动栈，投入产出比极低。`@tcp` 与 `@o2ib` 可以共存（多 rail），但那是生产优化范畴。
- **防火墙**：LNet socklnd 监听 **TCP 988** 端口（接收连接）。实验环境直接关 firewalld；若要放行，开放 `988/tcp` 即可（详见 §6 排查表）。

### 3.5 虚拟化平台注意事项

**通用（所有平台）**
- **时钟漂移是最大隐患**：Lustre 客户端与服务器之间的 RPC 有时间戳/恢复机制，时钟偏差会导致客户端被 `evicted`（报 `LustreError: ... timed out`、`no recovery`）。每个节点必须配置 **chrony/NTP** 指向同一时间源（见 §5.1 第 3 步）；
- **关闭 SELinux 或正确打标签**：实验环境建议 `SELINUX=disabled`（Lustre 内核模块与 SELinux 的标签体系兼容性差，尤其 `mount -t lustre` 时）；
- **避免内存气球（balloon）与 swap**：VM 内存气球回收、swap 抖动会让 Lustre 处理超时，进而触发客户端 evict。实验环境给足内存、关闭或调低 swap：
  ```bash
  swapoff -a && sed -i '/ swap /s/^/#/' /etc/fstab
  ```
- **CPU 绑核**：非必要。若宿主机多核空闲，可用 `virt-manager` 的 CPU pinning 或 `taskset` 绑核，减少抖动；单核过载是时钟/超时问题的常见诱因。

**KVM/QEMU**
- 网卡、磁盘都选 **virtio**（virtio-net / virtio-scsi）；
- 用 `virt-install` 或 virt-manager 创建，示例：
  ```bash
  virt-install --name mds1 --ram 8192 --vcpus 4 \
    --disk path=/data/vms/mds1.qcow2,size=20,format=qcow2,bus=virtio \
    --disk path=/data/vms/mds1-mgt.qcow2,size=1,format=qcow2,bus=virtio \
    --disk path=/data/vms/mds1-mdt.qcow2,size=10,format=qcow2,bus=virtio \
    --os-variant rhel8.10 --network bridge=br0,model=virtio \
    --cdrom /data/iso/Rocky-8.10.iso --boot cdrom,hd
  ```

**VMware（Workstation/Fusion/ESXi）**
- 磁盘控制器选 **PVSCSI**，网络选 **VMXNET3**（不要用 e1000）；
- VMXNET3 默认开启 **TCP offload**，个别内核/驱动组合下会导致大包丢包（Lustre 表现为随机超时）。若出现，用 `ethtool -K ens192 tx off rx off` 临时关闭验证；
- ESXi 中给 VM 禁用内存气球：`mem.hotadd` 相关设置，或直接给足预留内存。

**VirtualBox**
- 磁盘控制器用默认 SATA/AHCI，或 NVMe；
- VirtualBox 的虚拟网络（NAT 的 988 端口转发麻烦）**建议选 Bridged（桥接）**，所有 VM 与宿主机同网段、用静态 IP；
- 性能上限较低（单线程 I/O、无多队列），适合学习验证，不适合做性能测试；
- 注意 VirtualBox 默认不装 Guest Additions 时 virtio 网卡不可用，用默认的 Intel PRO/1000 即可。

---

## 4. 架构说明

### 4.1 单节点部署（学习/功能验证）

```
┌────────────────────────────── 一台 VM（192.168.10.11） ──────────────────────────────┐
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │
│  │ MGS + MGT     │  │ MDS + MDT     │  │ OSS + OST0    │  │ OSS + OST1    │        │
│  │ /dev/sdb      │  │ /dev/sdc      │  │ /dev/sdd      │  │ /dev/sde      │        │
│  │ /mnt/mgt      │  │ /mnt/mdt      │  │ /mnt/ost0     │  │ /mnt/ost1     │        │
│  └───────────────┘  └───────────────┘  └───────────────┘  └───────────────┘        │
│                            ↑ 同一台机器内部 LNet @tcp                               │
│  ┌───────────────┐  挂载 192.168.10.11@tcp0:/lustre → /mnt/lustre                   │
│  │ Client（本机） │                                                                  │
│  └───────────────┘                                                                  │
└────────────────────────────────────────────────────────────────────────────────────┘
```

- **适用**：快速跑通「元数据+数据+挂载」全链路、学习 `lctl`/`lfs` 命令、验证版本兼容性；
- **优点**：资源占用最小、排障路径最短（网络问题几乎为零）；
- **局限**：无法演示多节点网络通信与故障隔离。

### 4.2 多节点部署（推荐，贴近生产）

```
        Management/Data 网络：192.168.10.0/24（LNet @tcp，端口 988）
┌──────────────┐        ┌──────────────┐        ┌──────────────┐
│   mds1       │        │   oss1       │        │   client1    │
│ 192.168.10.11│        │ 192.168.10.12│        │ 192.168.10.13│
│ MGS + MGT    │        │ OSS + OST0   │        │ Client       │
│ MDS + MDT    │        │ OSS + OST1   │        │ /mnt/lustre  │
│ /dev/sdb=MGT │        │ /dev/sdb=OST0│        └──────┬───────┘
│ /dev/sdc=MDT │        │ /dev/sdc=OST1│               │
└──────┬───────┘        └──────┬───────┘               │
       │                       │                       │
       └──── 988/tcp  ◄────────┴──────── 988/tcp ◄─────┘
```
> 加一台 `oss2`（192.168.10.14，OST2/OST3）即可扩展为 4 节点，命令与 oss1 完全一致（换 `--index`）。

- **适用**：接近生产形态，可演示「MDS 故障客户端挂死」「OST 故障 IO 报错」「增加 OSS 扩容」；
- **优点**：命令与生产部署 1:1 对齐；
- **局限**：需要先排查纯网络问题（防火墙、NID 拼写、端口）。

### 4.3 单节点 vs 多节点的命令差异

| 环节 | 多节点 | 单节点 |
| --- | --- | --- |
| MGS NID 引用 | 写其他节点 IP：`--mgsnode=192.168.10.11@tcp0` | 写本机 IP（`--mgsnode` 可省，但建议仍写，保持命令一致） |
| `mount -t lustre` | `<MGS-IP>@tcp0:/lustre` | 同样写法，指向本机 IP |
| LNet 配置 | 每台机器各自 `lnetctl net add` 自己的网卡 | 只配一次 |
| 部署顺序 | 按 mds1 → oss1 → client1 顺序执行 | 同一台机器按 MGS→MDT→OST→Client 顺序执行 |
| 排障重点 | 防火墙/端口/主机名解析 | 磁盘设备名 |

---

## 5. 部署流程（以多节点为例，单节点差异见 §4.3）

> 以下所有命令默认在 **Rocky/Alma/RHEL 8.10 + Lustre 2.15.8（方案 A）** 上执行；方案 B（CentOS 7.9 + 2.12.9）把 `dnf` 换成 `yum`、仓库路径换成对应 el7 路径即可。文中 `ens192` 是示例网卡名，请以 `ip link` 为准。

### 5.1 环境准备（所有节点都要做）

```bash
# 1) 主机名（在每台节点上分别执行，填各自主机名）
hostnamectl set-hostname mds1
hostnamectl set-hostname oss1
hostnamectl set-hostname client1

# 2) /etc/hosts 全节点一致（Lustre 通信建议全走静态 IP，不依赖 DNS）
cat >> /etc/hosts <<EOF
192.168.10.11  mds1
192.168.10.12  oss1
192.168.10.13  client1
EOF

# 3) 时间同步（关键！全节点指向同一时间源，避免客户端被 evict）
dnf install -y chrony
systemctl enable --now chronyd
chronyc sources -v            # 确认同步状态；出现 ^* 即同步成功

# 4) 关闭 SELinux 与防火墙（实验环境）
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
systemctl disable --now firewalld
# 若不想关防火墙，至少放行 LNet 端口：
# firewall-cmd --permanent --add-port=988/tcp && firewall-cmd --reload

# 5) 基础工具
dnf install -y net-tools bind-utils iproute vim
```
> ⚠️ SELinux 修改后需**重启**生效；本节可以先做，重启放到 5.2 安装内核之后统一进行。

### 5.2 安装 Lustre 软件（服务端 mds1/oss1；客户端 client1 见 5.8）

```bash
# 1) 配置 Whamcloud 仓库（服务端节点；客户端可只配 client 段）
cat > /etc/yum.repos.d/whamcloud.repo <<'EOF'
[lustre-server]
name=Lustre Server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.8/el8.10/server/
enabled=1
gpgcheck=0

[lustre-client]
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

# 2) 安装依赖 EPEL
dnf install -y epel-release

# 3) 安装 Lustre 补丁内核（先查清仓库里确切的 kernel 包名/版本再装）
dnf list available 'kernel*' --enablerepo=lustre-server | grep lustre
dnf install -y kernel-4.18.0-553.82.1.el8_lustre \
               kernel-devel-4.18.0-553.82.1.el8_lustre --nogpgcheck

# 4) 把补丁内核设为默认启动项，并锁定内核版本（防止 yum update 换内核导致模块失效）
grubby --set-default /boot/vmlinuz-4.18.0-553.82.1.el8_lustre.x86_64
echo "exclude=kernel-4.18.0* kernel-core kernel-devel kernel-headers" >> /etc/dnf/dnf.conf

# 5) 安装 Whamcloud e2fsprogs（ldiskfs 后端必需）
dnf install -y e2fsprogs --enablerepo=e2fsprogs-wc --nogpgcheck

# 6) 安装 Lustre 服务端软件
dnf install -y lustre lustre-osd-ldiskfs-mount kmod-lustre kmod-lustre-osd-ldiskfs --nogpgcheck

# 7) 重启进入补丁内核，验证
reboot
uname -r                          # 期望：4.18.0-553.82.1.el8_lustre.x86_64
modprobe lustre && lsmod | grep lustre    # 无报错即为模块加载成功
lctl version                      # 期望输出 2.15.x 版本号
```
> ⚠️ 若 `modprobe lustre` 报错，先 `dmesg | tail -20` 看原因，排查方法见 §6.1。

### 5.3 配置 LNet（所有节点，含客户端）

LNet 支持两种配置方式，二选一（推荐方式 1）：

**方式 1：动态配置（2.10+ 推荐，`lnetctl`）**
```bash
# 启用 LNet 并添加网络；--if 填实际网卡名（ip link 查看）
lnetctl lnet configure
lnetctl net add --net tcp0 --if ens192
lnetctl net show                   # 查看网络状态，应显示 up
lctl list_nids                     # 期望：192.168.10.11@tcp0
```
**方式 2：静态 modprobe 参数（老版本/需要持久化时）**
```bash
cat > /etc/modprobe.d/lnet.conf <<EOF
options lnet networks=tcp0(ens192)
EOF
modprobe lustre                    # 重新加载模块使配置生效
lctl list_nids
```
> 若使用方式 1，请把 `lnetctl` 命令写入启动脚本或 systemd 服务以保证开机自启（见 §5.10）。

**连通性验证**（服务端之间、客户端到 MGS）：
```bash
lnetctl ping 192.168.10.11@tcp0    # 在任一节点 ping 其他节点；2.13 起 lctl ping 已并入 lnetctl
```

### 5.4 创建并挂载 MGS/MGT（mds1）

```bash
# MGT 很小（几 GB 足够），用一块独立盘 /dev/sdb
mkfs.lustre --fsname=lustre --mgs /dev/sdb
mkdir -p /mnt/mgt
mount -t lustre /dev/sdb /mnt/mgt
```
> `--fsname` 在 MGS 上可选，但写上保持配置自解释。格式化输出会提示 `Writing CONFIGS/mountdata`，即 MGT 已初始化。

### 5.5 创建并挂载 MDT（mds1）

```bash
# 单 MDT 必须 --index=0；--mgsnode 填 MGS 的 NID（单节点填本机 IP）
mkfs.lustre --fsname=lustre --mdt --index=0 \
    --mgsnode=192.168.10.11@tcp0 /dev/sdc
mkdir -p /mnt/mdt
mount -t lustre /dev/sdc /mnt/mdt
```

### 5.6 创建并挂载 OST（oss1，每个 OST 一块独立盘）

```bash
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
```

### 5.7 服务端验证

```bash
# 在 mds1 查看已加载的目标列表（MGS/MDT/OST 都应在）
lctl dl
# 期望看到多行，例如：
#   0 UP mgs MGS MGS 9
#   1 UP mdt MDS MDS_MDT0000 7 ...
#   2 UP osc lustre-OST0000-osc-MDT0000 ...  (OSS 注册后自动出现)

# 查看 MGS 上记录的文件系统配置（应能看到 lustre-MDT0000、lustre-OST0000/0001）
lctl get_param mgs.MGS.live.*

# 在客户端视角确认目标可访问（client1 上执行，先装客户端见 5.8）
lfs df                        # 挂载后可列出 MDT/OST 容量
```

### 5.8 客户端安装与挂载（client1）

```bash
# 1) 配置客户端仓库（只留 client 与 e2fsprogs 段即可），然后：
dnf install -y epel-release
dnf install -y lustre-client --nogpgcheck
modprobe lustre
lctl version                  # 客户端也要求版本与服务端一致

# 2) 配置 LNet（同 §5.3，方式一或方式二）
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
```

### 5.9 功能验证

```bash
# 1) 文件系统总览
lfs df -h                              # 应显示 MDT 与所有 OST 的容量

# 2) 查看默认布局与条带化
lfs getstripe /mnt/lustre              # 目录/文件默认布局（-c 0 表示用 fs 默认）
lfs setstripe -c 2 -S 4M /mnt/lustre/striped_test   # 建一个跨 2 个 OST、4MB 条带的文件
lfs getstripe /mnt/lustre/striped_test # 确认 obdidx 列显示 0,1（两个 OST）

# 3) 简单 IO 测试
dd if=/dev/zero of=/mnt/lustre/bigfile bs=1M count=1024 oflag=direct   # 写 1GB
dd if=/mnt/lustre/bigfile of=/dev/null bs=1M oflag=direct               # 读回
# 用 fio 更全面：
dnf install -y fio
fio --name=test --rw=rw --bs=1M --size=512M --numjobs=4 \
    --directory=/mnt/lustre --ioengine=libaio --direct=1

# 4) 文件在多个 OST 上真实分布
lfs getstripe /mnt/lustre/bigfile
```

### 5.10 开机自启与持久化

| 节点 | 需要持久化的内容 | 方法 |
| --- | --- | --- |
| 所有节点 | LNet 网络（方式 2 已随 modprobe 参数固化；方式 1 需手动） | 方式 1 用户：新建 `/etc/systemd/system/lnet-setup.service` 执行 `lnetctl lnet configure && lnetctl net add --net tcp0 --if ens192`，`systemctl enable lnet-setup` |
| mds1 | MGT/MDT 挂载 | fstab：`/dev/sdb /mnt/mgt lustre defaults,_netdev 0 0`（用 UUID 更稳，见 §6.9） |
| oss1 | OST 挂载 | fstab 同上 |
| client1 | 客户端挂载 | fstab：`...:/lustre /mnt/lustre lustre defaults,_netdev 0 0` |
| 旧式（el7） | — | `chkconfig lnet on && chkconfig lustre on`（老 init 脚本方式） |

> 注意：**服务端目标 fstab 不要用 `_netdev` 之外的其他挂载选项**（如 `noatime` 可加，但不要动 `defaults` 语义）；开机时 MGS 必须先于 MDT/OST 就绪，同机多个目标的 `_netdev` 会由内核自动等待。

---

## 6. 常见问题与排查

> 排障总原则：`dmesg | tail`、`journalctl -xe`、`/var/log/messages` 三个日志先行；其次 `lctl dl` 看目标状态；最后检查网络（`ping`、`nc -vz <ip> 988`）。

### 6.1 内核模块加载失败

| 现象 | 原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| `modprobe lustre` 报 `Invalid module format` | 内核头文件与运行内核不一致，或模块针对不同内核编译 | `uname -r`；`rpm -q kernel-devel` | `dnf install kernel-devel-$(uname -r)` 后重编模块；服务端必须用 Whamcloud 补丁内核 |
| 报 `Unknown symbol` / `no symbol version` | 模块依赖（如 `libcfs`、`lnet`）缺失或顺序错误 | `modprobe libcfs; modprobe lnet; modprobe lustre`（按序手动试） | `depmod -a` 重建依赖；`lsmod` 检查是否已加载同名旧模块 |
| 报依赖包缺失（找不到 `e2fsprogs` 特有版本） | 没装 Whamcloud e2fsprogs | `rpm -q e2fsprogs` | 启用 `e2fsprogs-wc` 仓库重装 |
| 模块加载了但 `lctl version` 报错 | 用户态工具与内核模块版本不匹配 | `rpm -q lustre kmod-lustre` | 统一升/降到同一版本号 |

### 6.2 时间不同步导致客户端被 evict

| 现象 | 原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| 客户端频繁报 `LustreError: ... timed out`、`recovering`、`connection to ... was lost`；`dmesg` 出现 `evicted` | 节点时钟偏差超过恢复窗口，服务端判定客户端"失联"并驱逐 | `timedatectl`；`chronyc sources -v`；服务端 `lctl get_param mdt.*.exports.*.client_*` | 全集群用 chrony 对齐同一时间源；被 evict 后 `umount` 再 `mount` 恢复 |
| 挂载后 30~90 秒无操作即报超时 | NTP 服务未启动或指向不同源 | `systemctl status chronyd` | 修正配置后重启 `chronyd` |

### 6.3 LNet 配置错误

| 现象 | 原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| `lctl list_nids` 为空 | LNet 未配置/未 up | `lnetctl net show` | `lnetctl lnet configure` 后 `lnetctl net add --net tcp0 --if <ifname>` |
| `lnetctl ping` 不通 | NID 拼写/网卡名/网段错误 | `ip a`；`lnetctl net show --detail` | 核对网卡名与 IP；`--if` 填实际接口 |
| `lnetctl ping` 通但 `mount` 失败 | 防火墙挡 988/tcp | `nc -vz <ip> 988` | 放行 `988/tcp` 或关闭 firewalld |
| `Connection timed out` 偶发 | 网卡 offload 导致大包丢包（VMware 常见） | `ethtool -k <if> tx-checksum-ipv4` 等 | `ethtool -K <if> tx off rx off` 验证；或换 virtio/vmxnet3 驱动参数 |
| NID 写成了 `@tcp1` 而实际是 `@tcp0` | LNet 网络名不一致 | `lctl list_nids` 对比两端 | 全集群统一网络名（默认 `tcp0`） |

### 6.4 挂载失败

| 现象 | 原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| `mount.lustre: Connection timed out` | MGS 不可达/未启动/防火墙 | `lnetctl ping <mgs-nid>`；`nc -vz <mgs-ip> 988` | 启动 MGS（`mount -t lustre /dev/sdb /mnt/mgt`）；放行端口 |
| `mount.lustre: not a Lustre filesystem` | 设备不是 Lustre target，或磁盘已被破坏 | `mkfs.lustre --dryrun /dev/sdX`；`dumpe2fs /dev/sdX \| head` | 用 `mkfs.lustre` 重新格式化（注意数据） |
| `mount.lustre: Can't contact MGS` | 服务端 LNet 未起来，或 `--mgsnode` 写错 | 在 MGS 上 `lctl list_nids` | 修正 MGS NID 后 `umount`/`mount` 重试 |
| `mount.lustre: mount option error` | fstab 选项写错（如 `defaults,_netdev` 写成 `defaults_netdev`） | `grep lustre /etc/fstab` | 改回 `defaults,_netdev` |

### 6.5 OST 未注册 / 看不到

| 现象 | 原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| `lctl dl` 中 OST 状态为 `IN`（inactive）或缺失 | OST 未挂载、或 MGS 尚未下发配置 | `lctl dl`；OSS 上 `mount -t lustre /dev/sdX /mnt/ostX` | 在 OSS 上挂载该 OST；等待数秒后客户端 `lctl dl` |
| 新增 OST 后 `lfs df` 不显示 | 客户端缓存了旧配置 | `lctl dl`（客户端）确认 osc 存在 | `umount`/`mount` 客户端；或 `lctl get_param mgs.MGS.live.*` 核对 MGS 记录 |
| `mkfs.lustre --ost --index=N` 报 index 已占用 | 索引冲突（不同 OST 同 index） | `lctl get_param mgs.MGS.live.*` | 重新指定未占用 index；确认后重新格式化 |
| OST 挂载后随即 `LustreError: ... expired` | 生成代次（generation）与 MGS 记录不符（如做过快照回滚） | 服务端日志找 `generation` 字样 | 官方做法 `mkfs.lustre --writeconf` 重置（需谨慎，会丢弃该 target 的旧布局配置） |

### 6.6 磁盘设备名漂移

| 现象 | 原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| 重启后 fstab 里的 `/dev/sdb` 变成了别盘，挂到错误 target | VM 磁盘枚举顺序在重启后变化 | `lsblk -f`；`ls -l /dev/disk/by-id/ /dev/disk/by-uuid/` | 用 `UUID=` 或 `/dev/disk/by-id/...` 替换 fstab 与 mount 命令；**格式化后立刻记录 UUID** |
| `mkfs.lustre` 把数据盘当系统盘格式化 | 设备名看错 | `lsblk` 核对容量与挂载点 | 格式化前再三核对设备；用 by-id 路径 |

### 6.7 虚拟化特有

| 现象 | 原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| 所有节点时间稳定但客户端仍周期性 evict | 时钟漂移叠加 VM 暂停/迁移 | `chronyc tracking` 看 offset | 校正时间源；避免 VM 挂起（suspend） |
| 快照回滚后集群集体挂不上 | MGT/MDT/OST 与 MGS 配置代次不一致 | `dmesg` 报 `CONFIG gen ... mismatch` | 无快照回滚；只有整机重做 target 或 `--writeconf` |
| 性能忽高忽低 | 磁盘稀疏文件扩展、内存气球、宿主机争抢 | `iostat -x 1`；`free -h` | 换 thick 盘、给足内存、错峰实验 |
| VirtualBox 下 `lnetctl ping` 不通 | 网络模式非桥接，或 VM 间不可达 | `ping <ip>`；`ip route` | 改用 Bridged，静态 IP |

---

## 7. 附录

### 7.1 快速核对清单（Checklist）

**部署前**
- [ ] 已确认发行版 + Lustre 版本 + 内核版本三者匹配（§2.1 表）
- [ ] 已登录 downloads.whamcloud.com 核对仓库中确切的 kernel/包版本号
- [ ] 每台 VM 静态 IP、`/etc/hosts` 全集群一致
- [ ] 所有节点 chrony 已指向同一时间源并 `chronyc sources` 显示同步
- [ ] SELinux 已 disabled（或已规划放行规则）；firewalld 已关闭或放行 988/tcp
- [ ] MGT/MDT/OST 各用独立虚拟盘，已记录各盘 UUID
- [ ] 明确告知团队：**不做 VM 整机快照回滚**

**部署中（按序）**
- [ ] 服务端补丁内核安装 → `grubby --set-default` → 重启后 `uname -r` 正确
- [ ] `dnf.conf` 已加 `exclude=kernel*` 锁定内核
- [ ] Whamcloud e2fsprogs 已安装
- [ ] `modprobe lustre` 无报错，`lctl version` 版本正确
- [ ] LNet 已 up，`lctl list_nids` 输出正确 NID，`lnetctl ping` 全互达
- [ ] MGS/MGT 挂载成功；MDT(index 0) 挂载成功；OST(index 0,1,...) 挂载成功
- [ ] mds1 上 `lctl dl` 显示 MGS/MDT/OST 全部 UP
- [ ] 客户端 `mount -t lustre <mgs>:/<fs> /mnt/lustre` 成功
- [ ] `lfs df -h` 显示 MDT 与全部 OST
- [ ] `lfs setstripe -c 2` + `dd` 1GB 读写通过
- [ ] fstab 已写入（服务端目标 + 客户端挂载），`mount -a` 复验

**部署后**
- [ ] 执行一次整机重启，确认 LNet、所有目标、客户端挂载自动恢复
- [ ] 备份方式已改为 Lustre 原生手段（非 VM 快照）

### 7.2 参考资料

| 资料 | 地址 | 用途 |
| --- | --- | --- |
| Lustre 官方 Wiki | https://wiki.lustre.org/ | 安装/运维文档首页 |
| Lustre 安装软件指南 | https://wiki.lustre.org/Installing_the_Lustre_Software | 仓库配置官方依据 |
| KVM 快速上手 | https://wiki.lustre.org/KVM_Quick_Start_Guide | KVM 虚拟化部署参考 |
| Whamcloud 下载站 | https://downloads.whamcloud.com/ | 所有预编译包/SRPM |
| 支持矩阵 | https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix | 版本×发行版×内核权威对照 |
| Lustre 官方手册 | https://wiki.lustre.org/Lustre_Manual | 概念、mkfs.lustre/lctl 全参数 |
| 本文档配套 | `docs/Lustre_Manual_cn_0.0.3.pdf` | 官方手册中文版，离线查阅 |
