# Slurm 集群现场交付上架与现场适配指南（aiclust · 10 节点）

> **用途**：交付工程师到**客户现场**把 aiclust（master + storage + gpu1~gpu8，10 节点）上架、开电、改 IP、验收、移交的**全流程执行手册**。
> **一句话总结"到底改哪些地方"**：本集群跨节点引用**绝大多数走主机名靠 `/etc/hosts` 解析**（不走 IP），所以——
> - **只改 IP、不改主机名**：真正要动手的只有 **3 处**（①10 台 `/etc/hosts` ②各台 `/etc/slurm/slurm.conf` 的 `NodeAddr` ③master `/etc/chrony/chrony.conf` 的 `allow` 网段）；若现场要开防火墙再多放行一组端口。
> - **连主机名也改**：改动面扩大（§5.2），且会波及 Slurm 记账的节点名/集群名。
> - **现场要启用 IB 高速网络**：另需新增 **4 件事**（§5.3）——①IB 交换机/SM 确认与 opensm 子网管理器 ②10 台 `ib0` 的 IPoIB 静态 IP ③（可选）`/etc/hosts` 增加 IB 主机名 ④MPI/应用层走 IB 验证；Slurm 管理面**默认仍走业务网**，不改 `NodeAddr`。
> - 环境层（公司机房内网→客户机房）可能还要处理：**外网不通→换软件源/离线包、NTP 源变更、账密交接、RAID 硬核验**。
>
> 关联深读文档：《Slurm集群现场IP适配清单.md》（IP/主机名改动细节）、《aiclust集群交付手册-验收命令清单.md》（逐项验收命令）、《Slurm集群10节点手操部署文档.md》（需重装时）。

---

## 0. 现场前准备（行前必带，别等到现场）

| 类 | 内容 | 备注 |
| --- | --- | --- |
| 账密 | `ps` / `Puersai.168`（10 台默认一致） | 若现场给新密码，改后同步记录 |
| 工具 | `storcli`(/libs)、`megacli` 安装包；U 盘/ISO；网线/console 线；笔记本 + 网口 | RAID 硬核验**必须在现场做**（见 §3.5） |
| 资料 | 本指南 + 上面 3 份关联文档；静态 IP 分配表模板（§4.1） | — |
| 关键资产 | 机柜/上架位编号、客户侧联系人、IDC 网段与网关/DNS、是否 DHCP | 到现场前尽量拿到网段 |
| 软件源 | 若客户机房**不通外网**：预拷 Ubuntu apt 镜像/离线 debs 或内网镜像地址 | 否则 `apt-get` 不可用，见 §6.1 |
| IB 互连 | **IB 交换机**（确认型号/是否自带子网管理器 SM）、IB 线缆（QDR/HDR 对应）、`opensm`/`ibutils`/`perftest`/`rdma-core` 包；IB 网段规划（§4.1-B） | 现场启用 IB 必带，见 §5.3 |

> **提醒**：`munge.key`（10 台共享）与 IP/主机名、网段**无关**，现场**一律不要动**——换掉会让全集群 MUNGE 认证失效。`/etc/exports` 用 `*` 通配已对所有客户 IP 放行 NFS，**也不用改**。

---

## 1. 到货开箱点验（到现场第一步）

按《交付配置单》对 10 台逐一点验：

```
| 序号 | 节点名(标签)   | 主机型号/SN      | 关键硬件(RAID盘数/GPU数/内存/CPU核) | 电源×2 | 网口数 | 是否齐|
| ---- | ------------- | ---------------- | ----------------------------------- | ------ | ------ | ------ |
|  1   | master  (ctrl) |                  | CPU 64核 / 内存 251G / 无GPU          | ☐ ☐    |        | ☐     |
|  2   | storage       |                  | CPU 64核 / 251G / RAID12盘→220T       | ☐ ☐    |        | ☐     |
|  3-8 | gpu1~gpu8     |                  | CPU 384核 / 1TB / 8×A100 80G          | ☐ ☐    |        | ☐     |
```
- 标签/SN 与主机名对应关系逐一核对；缺少的现场记录并反馈。
- 核对每台 RAID 卡、SSD/HDD 盘数、GPU 卡数与配置单一致。

---

## 2. 上架 + 布线与加电

1. **上架**：10 台按规划放入机柜位（建议 master/storage 照顾管理网聚合，gpu1-8 集中）。
2. **布线**：业务口 + 管理口（若客户提供双网）接客户交换机；记下**每台接的交换机口号**。/etc/hosts 依赖的是**业务口 IP**，务必确认业务口是哪张网卡。
3. **加电点检**（各台）：
   ```bash
   ip link                              # 确认网卡 link up、拿到 IP（或待 §4 配静态）
   nproc; grep MemTotal /proc/meminfo   # CPU 核数与内存
   nvidia-smi -L || echo 无GPU            # gpu 台应 8 张 A100
   lsblk -o NAME,SIZE,TYPE,MOUNTPOINT   # storage 见 RAID 卷 /mnt/sdb
   ```
4. **BIOS 时间**：若差较大先校准进系统后由 chrony 修正；确认固件/时间源设置。

---

## 3. 加电后硬件健康核验（现场必做重点）

> 过去在公司机房只验证了软件层；**RAID 阵列与整机健康**是现场上线前必须打勾项。

### 3.1 时间
```bash
date; timedatectl | grep -E 'synchronized|Time zone|RTC'
# 期望：System clock synchronized: yes；RTC in local TZ: no
```

### 3.2 GPU
```bash
# gpu 台
nvidia-smi                    # 8 张 A100 80G、驱动一致(如 595.80)、Persistence Mode ON
nvidia-smi -q -d TEMPERATURE,POWER,ECC
```

### 3.3 磁盘/文件系统
```bash
# storage
df -hT | grep -E '/mnt/sdb|/shared'     # 220T/220T
lsblk | head -30                           # 12 盘 RAID 5 + OS 盘
```

### 3.4 网络连通（10 台互 ping + 与客户网关）
```bash
getent hosts master storage gpu1 gpu8     # hosts 解析（先按现有 hosts 或改后）
for n in master storage gpu1 gpu8; do ping -c1 -W2 $n >/dev/null && echo "$n ok" || echo "$n FAIL"; done
ip route                                  # 默认路由/网关可达
```

### 3.5 RAID 阵列健康（**现场必装工具核验**）
集群内未装 storcli/megacli，现场用工具确认 storage 的 220T RAID5（AVAGO MR9361-8i）：
```bash
# 安装 storcli 后：
storcli /c0 show                # 控制器状态 = OK
storcli /c0/v0 show             # VD Volume=220T 状态=Optl(Optimal)
storcli /c0/e0/sall show        # 每盘 State=Onln、Rv(重构中)=N、盘温/健康
```
> 也可经 IPMI：`ipmitool sdr list | grep -iE 'state|temp|status'`。**阵列任一盘 State 非 Onln / VD 非 Optimal 都不得交付。**

### 3.6 IB 网卡与驱动核验（现场启用 IB 必做）

```bash
# 每台（gpu 台重点）：
ibstat                       # Port 1 State: Active（插线+SM 后才是 Active；未配 SM 时为 Down 属正常）
ibv_devinfo -v | head -20    # ConnectX-6 / 100Gb/s（HDR）等设备与速率
lsmod | grep mlx5_core       # 驱动模块在位
cat /sys/class/infiniband/*/ports/1/link_layer   # 期望 InfiniBand（不是 Ethernet）
ip link show ib0             # IPoIB 接口存在（未配 IP 属正常）
```
> 公司机房此前仅验证**驱动/设备在位**；IB 网络（SM、IPoIB、链路 Active）**尚未配置**，属现场启用项（§5.3）。若 `ibstat` 报无设备/`mlx5_core` 未加载 → 先报修，不进入 IB 配置。

---

## 4. 网络 / IP 适配（交付改造核心）

### 4.1 先做"现场静态 IP 分配表"（客户网段）

> 无论只改 IP 还是连主机名都改，先填这张表（IDC 网段/网关/DNS、主机名方案给客户确认）：

| 节点 | 现场主机名 | 现场 IP | 网关 | DNS | 掩码 |
| --- | --- | --- | --- | --- | --- |
| master | (同 `master` 或客户规范) | | | | |
| storage | | | | | |
| gpu1 | | | | | |
| gpu2 | | | | | |
| gpu3 | | | | | |
| gpu4 | | | | | |
| gpu5 | | | | | |
| gpu6 | | | | | |
| gpu7 | | | | | |
| gpu8 | | | | | |

- **建议**：客户机房配**静态 IP**；若强制 DHCP，务必按 10 台 MAC 在交换机/DHCP 保留 IP，防续租漂移。
- 业务网段可能与公司原来 `192.168.18/19` 完全不同，**网关/DNS 全部按现场填**。

#### 4.1-B IB 网络 IP 分配表（现场启用 IB 必填）

> IB 网是**独立二层网络**，建议单独网段（与业务网不冲突即可），10 台各一个 `ib0` 地址。示例规划 `10.10.0.0/24`（可换成现场规范）：

| 节点 | IB 接口 | IB IP（示例 10.10.0.0/24） | 掩码 | 备注 |
| --- | --- | --- | --- | --- |
| master | ib0 | 10.10.0.10 | /24 | opensm 子网管理器候选 |
| storage | ib0 | 10.10.0.11 | /24 | |
| gpu1 | ib0 | 10.10.0.21 | /24 | |
| gpu2 | ib0 | 10.10.0.22 | /24 | |
| gpu3 | ib0 | 10.10.0.23 | /24 | |
| gpu4 | ib0 | 10.10.0.24 | /24 | |
| gpu5 | ib0 | 10.10.0.25 | /24 | |
| gpu6 | ib0 | 10.10.0.26 | /24 | |
| gpu7 | ib0 | 10.10.0.27 | /24 | |
| gpu8 | ib0 | 10.10.0.28 | /24 | |

- IB 网**不需要网关/DNS**（纯集群内部互连），`ib0` 不设默认路由。
- 若客户 IB 交换机为 **Mellanox/NVIDIA 托管交换机**，通常自带 SM，可省去 §5.3.2 的 opensm 部署；确认后记录。

### 4.2 场景判定
- **A. 只改 IP、主机名不变（推荐，改动最小）** → 见 §5.1，**只动 3 处**。
- **B. 主机名也按客户规范改** → 见 §5.2，改动面大、波及记账。

---

## 5. 现场适配的"到底改哪些地方"

### 5.1 仅 IP 变化（主机名不变，**只改 3 处**）

| # | 文件 | 在哪台改 | 怎么改 |
| --- | --- | --- | --- |
| ① | `/etc/hosts` | **全部 10 台** | 用 §4.1 现场 IP 表覆写全集；`getent hosts` 验证 |
| ② | `/etc/slurm/slurm.conf` 的 `NodeAddr=` | 全部 10 台 | 仅把 `NodeAddr=旧IP` 换成现场 IP；**`NodeName` 一行都不动** |
| ③ | `/etc/chrony/chrony.conf` 的 `allow` 网段 | 仅 **master** 1 台 | 旧网段 `192.168.18/19.0/24` → 现场网段；`systemctl restart chrony` |

```bash
# ① 每台覆写 hosts（示例，IP 换成现场实际）
cat >/tmp/hosts.new <<'EOF'
127.0.0.1   localhost
<现场master-IP>     master
<现场storage-IP>    storage
<现场gpu1-IP>       gpu1
<现场gpu2-IP>       gpu2
<现场gpu3-IP>       gpu3
<现场gpu4-IP>       gpu4
<现场gpu5-IP>       gpu5
<现场gpu6-IP>       gpu6
<现场gpu7-IP>       gpu7
<现场gpu8-IP>       gpu8
EOF
sudo install -o root -g root -m 644 /tmp/hosts.new /etc/hosts
getent hosts master storage gpu1 gpu8

# ② 每台改 NodeAddr（只改 IP，NodeName/分区/SlurmctldHost 主机名不动）
sudo sed -i -E 's#NodeAddr=[0-9.]+#NodeAddr=<现场该节点IP>#' /etc/slurm/slurm.conf
sudo grep -n 'NodeAddr' /etc/slurm/slurm.conf

# ③ master 只改 chrony allow 网段
sudo sed -i -E 's#allow [0-9.]+/[0-9]+#allow <现场网段>/24#' /etc/chrony/chrony.conf
sudo systemctl restart chrony

# 统一刷新
sudo systemctl daemon-reload
# master：把新配置热加载到 ctld
sudo scontrol reconfigure
```

**哪些"看起来该改、实际不用动"（核对即可，勿手贱）**：

| 文件（10 台） | 内容 | 改不改 |
| --- | --- | --- |
| `/etc/munge/munge.key` | 纯密钥，不含 IP/主机名 | **否**（动了全集群认证失效） |
| storage `/etc/exports` | `/mnt/sdb *(rw,...)` | **否**（`*` 已对任意现场 IP 放行） |
| 各客户端 `/etc/fstab` | `storage:/mnt/sdb ...` | **否**（主机名 storage，hosts 解析） |
| master `/etc/slurm/slurmdbd.conf` | `DbdHost=master StorageHost=localhost` | **否**（主机名 / localhost） |
| `/etc/slurm/slurm.conf` 的 `SlurmctldHost`、`Nodes=`、分区 | 均为主机名 | **否** |
| systemd override / tmpfiles / limits | 无 IP | **否** |
| jobs 输出目录 `/shared/slurm-logs` | 共享区，NFS 无关 IP | **否** |

> 只改 IP **不需**重装任何组件、不需 touch munge.key、不需重建记账库。

### 5.2 主机名也变（按客户规范，改动面大）

先做"旧主机名→新主机名→新 IP"映射表，再逐台：

| 文件 / 配置 | 改什么 |
| --- | --- |
| 每台 hostname | `hostnamectl set-hostname <新名>` |
| 每台 `/etc/hosts` | 新名 ↔ 新 IP 全集 |
| 每台 `/etc/slurm/slurm.conf` | `SlurmctldHost=<新master名>`；所有 `NodeName=<新名> NodeAddr=<新IP>`；`cpu`/`gpu`/`cpugpu` 分区 `Nodes=<新名列表>` |
| master `/etc/slurm/slurmdbd.conf` | `DbdHost=<新master名>` |
| 各客户端 `/etc/chrony/chrony.conf` | `server <新master名> iburst` |
| 各客户端 `/etc/fstab` | `<新storage名>:/mnt/sdb ...`（若用主机名） |

**记账适配（关键，易漏）**：主机名/集群名变化会影响 slurmdbd 里的节点名/集群名映射。二选一：
- **A. 清理重建（测试/验收阶段推荐）**：停 `slurmd→slurmctld→slurmdbd` → 清 `/var/spool/slurm/ctld` + 重建 `slurm_acct_db` 库 → 反序启动 → 重走 §8/§9 记账账号。
- **B. 保留**（升级生产环境）：`sacctmgr` 重命名/逐项更新 `AccountingStorageHost`、确认 `cluster_table.cluster_name`。

**身份变更注意**：若客户要求新建 Unix 用户（非 `ps`），在 10 台建 **uid 一致**的账号，并把 `/shared/scratch`、`/shared/slurm-logs` 属主指到新 uid；Slurm 内也重建对应 `sacctmgr user` 关联（见 §7）。

### 5.3 IB 高速网络现场适配（现场启用 IB 必做）

> 现状：10 台均装有 **ConnectX-6（100Gb/s HDR）**、`mlx5_core` 驱动在位，但 **IB 网络未配置**（无 SM、无 IPoIB、链路 Down）。以下按"接线上联 → 子网管理器 → IPoIB 配 IP → 主机名 → 验证 → 应用走 IB"顺序执行。

#### 5.3.1 布线与上联确认

1. 每台 IB 口（`ibstat` 中的 Port 1）经 IB 线缆接客户 **IB 交换机**（QDR/HDR 线对应，注意接口类型）。
2. 确认交换机型号：**托管交换机（Mellanox/NVIDIA）自带 SM** → 跳过 5.3.2；**无源/不支持 SM 的交换机** → 必须在集群内跑 opensm。
3. 插线后（即使无 SM）`ibstat` 的 `State` 仍为 `Down`，属正常；**有 SM 后应变 `Active`**。

#### 5.3.2 子网管理器（SM）——缺一不可

> IB 网络必须且仅需**一个 SM**。优先用交换机自带 SM（省事）；否则在 **master** 上跑软件 `opensm`：

```bash
# 在 master 上（若客户交换机无 SM）：
sudo apt-get install -y opensm           # 离线时用预拷 debs
sudo systemctl enable --now opensm
systemctl is-active opensm                # active
tail -30 /var/log/opensm.log              # 应出现 SM port guid / Subnet Up 等字样
```
- 若交换机自带 SM 且已 Active，**不要再跑 opensm**（双 SM 会冲突，需配 `priority` 或关其一）。
- 只有 1 台 SM 即可；生产建议 master 为主 SM、可加备（改 `opensm.conf` 的 `sm_priority`）。

**验证 SM 生效**（在任一台）：
```bash
ibstat | grep -A2 'Port 1'    # State: Active、Physical state: LinkUp → SM 已接管
ibswitches                    # 能看到交换机 LID（需 ibutils 包）
```

#### 5.3.3 IPoIB：10 台 `ib0` 配静态 IP

按 §4.1-B 的 IB 分配表，每台用 **netplan**（Ubuntu 22.04）配 `ib0`（示例 master=10.10.0.10）：

```yaml
# /etc/netplan/99-ib0.yaml（每台新建，IP 换成本机）
network:
  version: 2
  infinibands:
    ib0:
      addresses: [10.10.0.10/24]
      # mtu: 65520        # 可选：全集群统一开高 MTU（需交换机支持）；默认 2044 最稳
```
```bash
sudo netplan apply
ip -4 addr show ib0         # 应见 10.10.0.10/24
ping -c2 -I ib0 10.10.0.10  # 先 ping 自己验证接口
```
> 若 `ib0` 被 NetworkManager 抢占：`sudo nmcli dev set ib0 managed no && sudo netplan apply`。
> **注意**：IB 网**不要加默认路由**（不加 `routes`/`gateway4`），否则可能干扰业务网默认路由。

#### 5.3.4 （可选）`/etc/hosts` 增加 IB 主机名

不影响功能（MPI 可用 IP 直连），但便于记忆/排障。在**全部 10 台** `/etc/hosts` 追加：

```
10.10.0.10   master-ib
10.10.0.11   storage-ib
10.10.0.21   gpu1-ib
10.10.0.22   gpu2-ib
...（按 §4.1-B 逐台对应）
```
> 仅追加，**不要改动**业务网 hosts 原有行（§5.1 ①）。

#### 5.3.5 验证 IB 全网连通

```bash
# 1) 链路与速率（各台）
ibstat | grep -E 'State|Rate'            # Active / 100 Gb/s (4X HDR) 等

# 2) IPoIB ping（master → 各 gpu）
for i in 10.10.0.2{1..8}; do ping -c1 -W1 -I ib0 $i >/dev/null && echo "$i ok" || echo "$i FAIL"; done

# 3) RDMA 数据面（perftest 包；需先在 target 起 server）
#    在 gpu2:  ib_write_bw -d mlx5_0        （默认监听 40000）
#    在 gpu1:  ib_write_bw -d mlx5_0 10.10.0.22
#    期望带宽接近 100Gb/s（如 90+ Gb/s）

# 4) MPI 应用走 IB（示例，openmpi 已装时）
mpirun --host gpu1,gpu2 -n 2 --mca pml ucx --mca btl ^openib \
  hostname                              # 或默认 btl 自动选 IB
ucx_info -d | grep -iE 'ib|rdma'        # 确认 UCX 检测到 IB 传输
```

#### 5.3.6 Slurm 管理面是否走 IB（决策点）

| 方案 | 做法 | 适用 |
| --- | --- | --- |
| **A. 管理面走业务网（推荐）** | `NodeAddr` 保持业务 IP **不动**；IB 仅服务 MPI/应用数据面 | 稳定性优先，IB 链路故障不影响调度 |
| B. 管理面走 IB | 各台 `NodeAddr=` 改 IB IP + `scontrol reconfigure` | 业务网带宽不足时；**需 IB 全链路先验收通过**，风险自负 |

> 本指南默认 **A**。若选 B，`/etc/slurm/slurm.conf` 的 `NodeAddr` 全部换 IB IP，且 **IB 故障时集群调度将中断**，务必评估。

#### 5.3.7 防火墙与 IB

- 若客户要求开 ufw/防火墙：IB 网为**内部互信网络**，建议**对 `ib0` 网段整体放行**（`sudo ufw allow from 10.10.0.0/24`）或 IB 网不启用防火墙。
- RDMA/IPoIB 无需额外开放 TCP/UDP 端口；放行 ICMP 便于 ping 排障。

---

## 6. 环境层适配（客户机房的环境差异）

### 6.1 外网不通 → 软件源/离线包
```bash
# 若有内网镜像： apt sources 指向内网镜像
# 若无外网：预拷 Ubuntu 22.04 debs / 离线 apt 包，用 dpkg -i ./*.deb 或设置本地源
# 需确保的包：chrony nfs-kernel-server nfs-common munge libmunge* build-essential libhwloc* libnuma1 libdbus* libbpf* libelf* libmariadb* mariadb-server hwloc 等
```

### 6.2 NTP 源变更
- 若客户机房隔离公网，master 的 chrony `pool ntp.ubuntu.com` 会失效 → 改为 `server <客户时间服务器>`，或 `local stratum 10` + 手工基准，客户端默认跟 master 不变。改后 `chronyc sources` 确认 `^*`。

### 6.3 防火墙 / 安全策略（现场若需开启 ufw/客户防火墙）
集群互信端口（按需放行，通常仅集群内部 + 管理口）：
```
6817 TCP   slurmctld 通信
6818 TCP   slurmd    通信（slurmd 注册/作业分派）
6819 TCP   slurmdbd  记账
munge     UDP 高位   认证
2049 + rpcbind TCP/UDP   NFS（storage → 各客户端）
123       UDP       NTP（chrony）
22        TCP       SSH 管理
```
若要限制外部，只把 **22 管理口**暴露给客户运维网段，集群互信端口可只在节点间内网放行。

### 6.4 账号/密码交接
现场给出后若客户要求改 `ps` 密码：`passwd ps`（10 台一致）；改了记得**更新**：su/sudo 密码（deploy 场景用）、SSH 免密不受影响（公钥不变）。生产环境建议将 SaaS/交付文档中的口令替换后重新归档。

---

## 7. 软服务自检（适配后，各台 + master）

```bash
# 每台
systemctl is-active munge slurmd chrony      # gpu/storage 应 active
df -hT | grep /shared                          # 220T 挂载
sha256sum /etc/munge/munge.key | awk '{print $1}'   # 10 台一致
echo go | munge -n | unmunge 2>&1 | grep -i success

# master
systemctl is-active slurmctld slurmdbd mariadb
sacctmgr -n show assoc format=Cluster,Account,User,DefaultAccount,QOS   # 记账账号仍 ps→aiclust
scontrol ping; sinfo                          # 10 节点 idle
# IB（现场启用时）
ibstat | grep -E 'State|Rate'; ping -c1 -I ib0 <gpu1-IB-IP>   # Active / 100Gb/s / 通
```

---

## 8. 上架后全面验收（达标才移交）

在 **master** 上按《aiclust集群交付手册-验收命令清单.md》逐项跑；核心抽查：
```bash
sinfo && scontrol ping
srun -p cpu -N2 -n2 hostname
srun -p gpu --gres=gpu:1 -w gpu1 nvidia-smi --query-gpu=index,name --format=csv,noheader
srun -p cpugpu -n 300 hostname
sbatch --wrap='sleep 2'; sleep 3
sacct -S now-1hour --format=JobID,State,Elapsed,NodeList,AllocTRES%20 | tail -5
sudo mysql -e "select count(*) from slurm_acct_db.aiclust_job_table;"
# IB（现场启用时加验）：
ibstat | grep -E 'State|Rate'                                  # 任一台：Active / 100 Gb/s
for i in 10.10.0.2{1..8}; do ping -c1 -W1 -I ib0 $i >/dev/null && echo "$i ok" || echo "$i FAIL"; done
```
期望：全节点 `idle`、作业 `COMPLETED`、`sacct` 含 billing/cpu/mem、记账表有条目；IB 各台 `Active`、IPoIB 全通。

**验收记录表**（交付归档）：

| 检查项 | 命令 | 应见 | 结果 | 现场签字 |
| --- | --- | --- | --- | --- |
| 10 节点就绪 | `sinfo` | 10 idle、三分区 | ☐ | |
| RAID 健康 | `storcli /c0/v0 show` | VD=Optl、盘=Onln | ☐ | |
| GPU | `nvidia-smi` | 8×A100、驱动一致 | ☐ | |
| NFS | `df -h | grep /shared` | 220T | ☐ | |
| 记账 | `sacct` + `mysql count` | COMPLETED + 有条目 | ☐ | |
| 时间 | `timedatectl` | synchronized、UTC | ☐ | |
| IB 链路 | `ibstat` | Port State=Active、Rate=100Gb/s | ☐ | |
| IB 连通 | `ping -I ib0 <各gpu-IB-IP>` | 8 台全通 | ☐ | |
| IB 带宽（可选） | `ib_write_bw` | 接近 100Gb/s | ☐ | |

---

## 9. 现场问题速查（上架/适配易错）

| 现象 | 原因/处理 |
| --- | --- |
| 互相 ping 不通 | hosts 未覆盖新网段 / 网卡口接错 / 交换机口未配 / 网关 |
| `slurmd` 起不来 | slurm.conf `NodeAddr` 还是旧 IP / hosts 与 NodeName 不匹配；改后 `scontrol reconfigure` |
| `sinfo` 节点 down | hosts 或时间不一致；先 `chronyc sources` |
| 时间不同步 | chrony `allow` 没改到现场网段 / 客户时间服务器不可达 |
| MUNGE 认证 fail | 误动了 munge.key（重启回滚备份） / 时间差 |
| `/shared` 不可见 | 防火墙 2049/rpcbind / fstab 主机名未解析 / storage `exportfs -ra` |
| 客户网段与旧网段冲突复用 | 确认 hosts + NodeAddr + chrony allow 三处全改 |
| `sacctmgr` 改了机器名 | 按 §5.2 记账适配 A/B 处理 |
| `ibstat` State 一直 Down | 线/交换机口/无 SM；先确认交换机型号，无 SM 则跑 opensm（§5.3.2） |
| 双 SM 冲突（opensm 报错/链路抖动） | 交换机自带 SM 时不要跑 opensm；或改 `opensm.conf` `sm_priority` |
| `ib0` 起不来 / 被 NetworkManager 接管 | `nmcli dev set ib0 managed no` 后 `netplan apply`（§5.3.3） |
| IPoIB ping 通但 MPI/带宽差 | 检查 `ib_write_bw` 走的是 `mlx5_0` 与正确对端；确认 MTU 全集群一致 |
| `ib_write_bw` 无服务监听 | 先在目标机起 server（`ib_write_bw -d mlx5_0`），再在发起端指定 IP |
| IB 配了默认路由导致业务网异常 | 移除 `ib0` 的 `routes`/`gateway4`，IB 网不设默认路由（§5.3.3） |

---

## 10. 交付物清单与移交

1. 上架/布线完成、10 台上电、静态 IP 已配。
2. RAID/GPU/容量健康已核验（§3）。
3. IP 或主机名已按 §5 适配、`scontrol reconfigure` 完成。
4. IB 网络已按 §5.3 启用（SM + IPoIB + 连通 + 带宽验证）。
5. 全量验收通过（§8 打勾）。
6. 移交资料：本指南 +《Slurm集群现场IP适配清单》+《aiclust集群交付手册-验收命令清单》+《Slurm集群10节点手操部署文档》+ 现场 IP 分配表 + IB 分配表 + 账密交接单。
7. 双方签字验收；记录 `storage` RAID 阵列状态、GPU ECC、现场时间服务器、IB 交换机/SM 配置等现场实况回填。

---

*现场工作流：点验 → 上架布线(业务+IB) → 加电健康核验(含 RAID、IB 卡) → 静态 IP 分配表 + IB 分配表 → IP/主机名适配(§5 三处 or 全量) → IB 网络启用(§5.3) → 环境适配(源/NTP/防火墙/账密) → 软服务自检 → 全量验收(含 IB) → 移交。改动前一律 `cp xxx xxx.orig` 备份。*