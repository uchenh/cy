# Lustre 单机测试机部署方案（Ubuntu 24.04 + Lustre 2.17.0 + ZFS）

> 面向"借一台物理测试机部署 Lustre"的场景。结论先行:**可以单机部署**。Lustre 官方支持 All-in-One 形态（MGS/MDS/OSS/Client 全部跑在同一台机器上），流程与 3 节点部署（`docs/06`）完全同构，只是所有角色合并到一台。适合功能验证、命令练习、给同事演示；**不适合性能测试与 HA 演练**（单机无法体现并行扩展，也没有故障隔离）。
>
> 本文所有命令与 06 实测环境（Ubuntu 24.04 + Lustre 2.17.0 + ZFS 2.2.2 + 6.8.0-138-generic）严格对齐，可直接照做；详细方法论见 `docs/05`，单机配置依据见 `docs/04` §3。

---

## 目录

1. [结论先行：一台能不能部署](#1-结论先行为什么可以单机部署)
2. [生产实际配置](#2-生产实际配置)
3. [单机 vs 3 节点：差异对照](#3-单机-vs-3-节点差异对照)
4. [部署步骤（全在一台机器）](#4-部署步骤全在一台机器)
5. [部署后验证](#5-部署后验证)
6. [单机局限与替代方案](#6-单机局限与替代方案)
7. [常见问题与排障](#7-常见问题与排障)
8. [参考信息](#8-参考信息)

---

## 1. 结论先行：为什么可以单机部署

Lustre 的组件（MGS/MDS/OSS/Client）是**逻辑角色而非物理要求**，官方 Quick Start 本身就支持单机部署。角色合一后：

- 一个节点同时承担 MGS、MDS、OSS、Client 四个角色，各自加载对应内核模块即可共存
- 数据仍走本地回环（本机 LNet），条带化、锁、挂载等**功能与多节点完全一致**
- 区别仅在：没有网络延迟/带宽体现，看不到横向扩展的聚合吞吐

**适用**：功能验证、命令学习、给同事演示、跑通全流程后再上多节点。
**不适用**：性能基准（单机带宽就是上限）、HA/故障切换演练、超大规模并发。

---

## 2. 生产实际配置

> 本节按**生产实际标准**给配置，不是"能跑"的最低配置。单机 All-in-One 本身是学习/验证形态，但若希望这台测试机跑出的结果具备生产参考价值，硬件就应按生产规格给；若借到的机器达不到，部署仍可完成，但性能/并发结果仅作参考，不具生产代表性。

| 项 | 生产实际配置 | 生产依据 |
|----|--------------|----------|
| CPU | **8~16 核，主频 ≥ 3.5 GHz** | MDS 元数据操作（create/lookup）吃单线程延迟，OSS 数据路径吃多核并行；高主频同时缩短服务端编译时间 |
| 内存 | **32~64 GB ECC** | ZFS ARC 默认上限为 50% 内存；还需 MDT inode 缓存与客户端页缓存。生产经验：每 1 TB OST 容量约需 ≥ 1 GB ARC，MDT 另需 4~8 GB 起步 |
| 系统盘 | **480 GB SSD**（SATA/NVMe 均可） | 系统 + 编译产物；生产建议双盘 RAID1 |
| MGT | **16 GB NVMe，独立盘** | 容量小，但它是集群挂载的前提，必须低延迟、高可靠（生产上 MGT 通常独立部署） |
| MDT | **500 GB~1 TB NVMe** | 元数据是随机 4K 小 IO，必须 NVMe 级延迟；容量按 inode 数估算，经验值约为后端总数据量的 1/100~1/1000 |
| OST | **2~4 块 × 2~4 TB，ZFS mirror/RAID-Z 或 HDD RAID5/6** | 生产单 OST 常见 4~16 TB（见 01 文档 §5.3）；**绝不做单盘**——无冗余，生产底线是 mirror 或 RAID-Z |
| 网络 | **10 GbE 起步，优先 RoCE/InfiniBand** | 生产 HPC 存储网标配 25~100 GbE；网络带宽必须匹配后端聚合吞吐，否则 IO 卡在网络上 |
| 其他 | ECC 内存、双电源/UPS、chrony/NTP 时间源 | 生产服务器硬件基线；时钟漂移会直接导致客户端被 evict |

**磁盘规划（生产口径）**：

- 每类 target 独立物理盘/独立 RAID 组，一 target 一盘（组）最直观，便于容量与故障隔离
- MDT 一定用 NVMe；OST 用 HDD RAID5/6（大容量）或 SSD（高性能），**后端必须带冗余**
- 若机器只有一块大盘：分区建池可以做功能验证，但属于降配，冗余与延迟都不达生产标准

**生产形态参考（多节点，真正的生产部署）**：

| 角色 | 生产配置 | 说明 |
|------|----------|------|
| MGS+MDS（2 台 active/passive） | 16 核 / 64 GB，MDT 用 NVMe 共享存储或 DNE 多 MDT | HA 必备；Pacemaker 故障切换 |
| OSS（按吞吐目标 N 台） | 16 核 / 64~128 GB，每台 2~4 OST × 4~16 TB（RAID） | 聚合带宽 ≈ OST 数 × 单盘带宽，按目标吞吐反推台数 |
| 网络 | RoCE/InfiniBand 25~100 GbE，管理网与存储网分离 | LNet 多 NI 隔离 |

> 说明：单机 All-in-One 无法满足生产的**高可用与组件隔离**要求（单点、资源共享），真正的生产部署必须是多节点形态。借机单机部署的意义在于：用接近生产的配置，验证架构与命令流程，为后续生产集群铺路。

---

## 3. 单机 vs 3 节点：差异对照

| 维度 | 3 节点（06 实测） | 单机（本文） |
|------|-------------------|--------------|
| 角色分布 | mds1(MGS+MDS)、oss1(OSS)、client1 | 全部合一（一台机器） |
| 执行机器 | 分散在 3 台 | 全部在同一台，无跨节点步骤 |
| OST index | 0、1 | **仍从 0 开始**（MDT=0，OST=0、1） |
| LNet | 三机互 ping | 只需本机 NID |
| /etc/hosts | 三机映射 | 本机即可（测试可不加） |
| 时间同步 | 三台 chrony | 一台 chrony（仍建议，防时钟漂移） |
| 客户端 | client1 单独装 DKMS 包 | **服务端 deb 已含客户端模块**，本机直接 `mount` |
| 编译 | mds1/oss1 各编一遍 | 只编一遍（省 40 分钟） |
| 自启 | fstab + cachefile + lnet-setup | 相同，但客户端挂载建议用等待式 systemd（06 §3.15 经验） |

> 一句话：单机部署 = 06 文档去掉"多机"两个字，命令几乎照抄。

---

## 4. 部署步骤（全在一台机器）

> 以下 `<测试机>` 表示在这台机器上执行。假设：登录用户 `user`，网卡 `ens33`，数据盘 `/dev/sdb`(MGT)、`/dev/sdc`(MDT)、`/dev/sdd`、`/dev/sde`(OST×2)。按实际替换。

### 4.1 环境探查与内核修正 `[测试机]`

```bash
cat /etc/os-release | head -3
uname -r                          # 若为 7.0.0-xx-generic（HWE 线），执行下方内核修正
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE

# 内核修正（仅当 uname -r 为 7.0 HWE；与 06 §3.2 相同）
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-138-generic"/' /etc/default/grub
sudo update-grub
sudo reboot
uname -r                          # ✔ 期望 6.8.0-138-generic
sudo apt-mark hold linux-image-generic linux-headers-generic
sudo apt purge -y linux-image-7.0.0-29-generic linux-headers-7.0.0-29-generic \
  linux-image-generic-hwe-24.04 linux-headers-generic-hwe-24.04 \
  linux-modules-7.0.0-29-generic
```

### 4.2 基础环境 `[测试机]`

```bash
sudo hostnamectl set-hostname lustre-single
sudo apt install -y chrony
sudo systemctl enable --now chrony
sudo apt install -y git curl wget ca-certificates gnupg lsb-release software-properties-common \
  build-essential gcc make linux-headers-$(uname -r)
# 测试环境建议关闭/调低 swap（防超时 evict，见 04 §3.5）
sudo swapoff -a && sudo sed -i '/ swap /s/^/#/' /etc/fstab
```

### 4.3 ZFS 与编译依赖 `[测试机]`

```bash
sudo apt install -y build-essential gcc make flex bison pkg-config \
  zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
  libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
  python3-dev dpkg-dev \
  libzfslinux-dev module-assistant debhelper quilt
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs && echo ZFS_LOADED
zpool version                     # ✔ 期望 zfs-2.2.2-0ubuntu9.4
```

### 4.4 服务端源码编译 `[测试机]`（约 40 分钟，后台）

```bash
export http_proxy=http://192.168.12.187:7897 https_proxy=http://192.168.12.187:7897
git clone --depth 1 https://github.com/lustre/lustre-release.git
cd ~/lustre-release
git fetch --depth 1 origin b2_17
git checkout -b b2_17 FETCH_HEAD
git log --oneline -1               # ✔ 期望 4b0407a New release 2.17.0

sh autogen.sh
./configure --enable-server --with-zfs --disable-ldiskfs \
    --with-linux=/usr/src/linux-headers-$(uname -r)
# ✔ 成功标志：结尾 "Type 'make' to build Lustre." + "checking if ZFS has ... yes"

nohup make debs -j4 > /tmp/lustre_make.log 2>&1 &
# 轮询：tail /tmp/lustre_make.log；产出 7 个 .deb
ls ~/lustre-release/debs/*.deb
```

### 4.5 安装模块并配置 LNet `[测试机]`

```bash
cd ~/lustre-release/debs
sudo dpkg -i *.deb
sudo depmod -a
sudo modprobe libcfs && echo LIBCFS_OK
sudo modprobe lustre && echo LUSTRE_OK
lctl version                      # ✔ 期望 2.17.0

# LNet：本机网卡 ens33；单机 NID 即本机 IP
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lctl list_nids               # ✔ 期望 192.168.x.x@tcp
```

> 单机部署服务端 deb 已包含客户端功能（llite/osc 模块），无需再装 Azure DKMS 包。

### 4.6 建池 + 创建 MGS/MDT/OST `[测试机]`

```bash
# 1) 四个池（MGT/MDT/OST0/OST1 各一，单盘单池）
sudo zpool create lustre-mgt /dev/sdb
sudo zpool create lustre-mdt /dev/sdc
sudo zpool create lustre-ost0 /dev/sdd
sudo zpool create lustre-ost1 /dev/sde
sudo zpool list                   # ✔ 四个池在线

# 2) mkfs：MGS → MDT(index=0) → OST(index=0,1)；--mgsnode 指向本机 NID
sudo mkfs.lustre --fsname=lustre --mgs --backfstype=zfs lustre-mgt/mgt
sudo mkfs.lustre --fsname=lustre --mdt --index=0 \
    --mgsnode=192.168.x.x@tcp --backfstype=zfs lustre-mdt/mdt
sudo mkfs.lustre --fsname=lustre --ost --index=0 \
    --mgsnode=192.168.x.x@tcp --backfstype=zfs lustre-ost0/ost
sudo mkfs.lustre --fsname=lustre --ost --index=1 \
    --mgsnode=192.168.x.x@tcp --backfstype=zfs lustre-ost1/ost

# 3) 挂载服务端目标
sudo mkdir -p /mnt/mgt /mnt/mdt /mnt/ost0 /mnt/ost1
sudo mount -t lustre lustre-mgt/mgt /mnt/mgt
sudo mount -t lustre lustre-mdt/mdt /mnt/mdt
sudo mount -t lustre lustre-ost0/ost /mnt/ost0
sudo mount -t lustre lustre-ost1/ost /mnt/ost1
mount | grep lustre               # ✔ 四行目标挂载

# 4) 服务端注册验证
sudo lctl dl                      # ✔ MGS / MDT0000 / OST0000 / OST0001 均 UP
sudo lctl get_param mgs.MGS.live.*   # ✔ 列出 lustre-MDT0000 / OST0000 / OST0001
```

> 单盘方案（只有一块大盘）时：`zpool create lustre-mgt /dev/nvme0n1p1`（分区）…… 各池建在不同分区上即可，mkfs/mount 命令不变。

### 4.7 客户端挂载（本机即是客户端）`[测试机]`

```bash
sudo mkdir -p /mnt/lustre
sudo mount -t lustre 192.168.x.x@tcp:/lustre /mnt/lustre
mount | grep lustre               # ✔ 出现 192.168.x.x@tcp:/lustre on /mnt/lustre
lfs df -h /mnt/lustre             # ✔ MDT0000 + OST0000 + OST0001
```

### 4.8 开机自启 `[测试机]`

```bash
# 1) fstab：服务端目标 + ZFS cachefile（客户端挂载不用 fstab，见下方说明）
# 追加：
#   lustre-mgt/mgt /mnt/mgt lustre defaults,_netdev 0 0
#   lustre-mdt/mdt /mnt/mdt lustre defaults,_netdev 0 0
#   lustre-ost0/ost /mnt/ost0 lustre defaults,_netdev 0 0
#   lustre-ost1/ost /mnt/ost1 lustre defaults,_netdev 0 0
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-mgt lustre-mdt lustre-ost0 lustre-ost1

# 2) LNet 自启（含 modprobe lnet 前置，避免 06 遗留的 failed）
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
WantedBy=multi-user.target
EOF
sudo systemctl enable --now lnet-setup

# 3) 客户端挂载用等待式 systemd（06 §3.15 经验：fstab _netdev 不等 MGS recovery，重启会丢挂载）
sudo tee /etc/systemd/system/lustre-client-mount.service >/dev/null <<'EOF'
[Unit]
Description=Mount Lustre client /mnt/lustre (waits for MGS)
After=network-online.target lnet-setup.service
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'for i in $(seq 1 30); do /usr/sbin/lnetctl ping 192.168.x.x@tcp >/dev/null 2>&1 && break; sleep 10; done'
ExecStart=/bin/mount -t lustre 192.168.x.x@tcp:/lustre /mnt/lustre

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now lustre-client-mount
```

---

## 5. 部署后验证

```bash
# 1) 容量总览：MDT + 2×OST
lfs df -h /mnt/lustre

# 2) 条带化功能验证（与 06 §3.12 相同）
sudo mkdir -p /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test
sudo dd if=/dev/zero of=/mnt/lustre/test/bigfile bs=1M count=512 oflag=direct conv=fdatasync
lfs getstripe -v /mnt/lustre/test/bigfile   # ✔ obdidx 含 0 和 1，条带化生效

# 3) 重启自愈：sudo reboot 后自动恢复（服务端目标 + 客户端挂载）
sudo systemctl is-active lustre-client-mount   # ✔ active
lfs df -h /mnt/lustre
```

**预期结果**：`lfs df -h` 显示 MDT0000 + OST0000 + OST0001（容量=各池实际容量之和）；条带文件横跨两个 OST；重启后全部自动恢复。

---

## 6. 单机局限与替代方案

| 局限 | 说明 | 替代/规避 |
|------|------|-----------|
| 看不到横向扩展 | 聚合带宽=单机上限，无法演示"加 OST 涨吞吐" | 性能演示仍需多节点 |
| MDS/OSS 资源竞争 | 元数据与数据 IO 抢同一 CPU/内存/磁盘 | 磁盘分开（MGT/MDT/OST 独立盘）已是最低要求 |
| 无故障隔离 | 单点，无法做 HA/切换演练 | 需要 HA 演练则用多机 |
| 锁争用被掩盖 | 单客户端/单机场景锁竞争不明显 | 多客户端并发测试需多机 |

**如果测试机配置较强（≥8 核 / 32 GB）**，两个进阶玩法：

1. **在测试机上起 3 台 VM 复刻 3 节点架构**（VMware/KVM，按 04 §3 配置）：相当于把"借来的机器"当宿主机，与 06 环境完全一致，还能演示跨节点网络路径
2. **把测试机并入现有集群当 OSS 节点**：如果目的是扩容量而非独立测试，直接按 `docs/08-lustre-新增OSS节点与数据盘.md` 场景 A 走（注意 OST index 取现有最大 +1）

---

## 7. 常见问题与排障

| # | 现象 | 根因 | 解决 |
|---|------|------|------|
| 1 | `./configure --with-zfs` 报 missing zfs development headers | 缺 ZFS 开发头文件 | `sudo apt install libzfslinux-dev` |
| 2 | `make debs` 报 Unmet build dependencies | 缺 module-assistant/debhelper/quilt | `sudo apt install module-assistant debhelper quilt` |
| 3 | `zfs-dkms` 安装卡死 | 在 7.0 HWE 内核上构建失败 | 先切 6.8 GA 并 purge 7.0 再装（§4.1）；卡死时 `pkill -f "dkms build -m zfs"` + `dpkg --configure -a` |
| 4 | GitHub clone 极慢 | 直连限速 | 走代理 `192.168.12.187:7897` |
| 5 | 本机 mount 报 EBUSY / 连不上 MGS | 重启后 MGS recovery 窗口期内客户端提前挂载 | 用 §4.8 等待式 systemd；`sudo lctl get_param mdt.lustre-MDT0000.recovery_status` 确认 COMPLETE |
| 6 | `lfs df` 只显示部分 OST | 有 OST 未挂载 | `mount \| grep lustre`、`lctl dl` 核对目标状态 |
| 7 | 单大盘分区部署，`zpool create` 报设备忙 | 分区被系统占用/挂载 | `lsblk` 确认分区未被挂载，卸载后重试 |
| 8 | 客户端被 evict | 时钟漂移 / 超时 | chrony 校时；swap 已关闭确认 |

---

## 8. 参考信息

| 资料 | 位置 / 地址 | 用途 |
|------|-------------|------|
| 本文（09） | `docs/09-lustre-单机测试机-部署方案.md` | 单机部署方案（本文） |
| 06 部署记录 | `docs/06-lustre-ubuntu24.04-3节点-部署方案.md` | 3 节点全流程（单机流程与之一致，命令照抄减多机） |
| 05 部署指南 | `docs/05-lustre-ubuntu24.04-部署指南.md` | 版本选型、后端选择、排障方法论 |
| 04 VM 指南 | `docs/04-lustre-虚拟机部署指南.md` | §3 单机配置建议、磁盘/网络/虚拟化注意点 |
| 08 扩容指南 | `docs/08-lustre-新增OSS节点与数据盘.md` | 若测试机并入现有集群当 OSS 节点 |
| Lustre 官方 Wiki | https://wiki.lustre.org/ | Quick Start / Building / 运维手册 |
| Whamcloud Support Matrix | https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix | 版本 × 发行版 × 内核 × ZFS 权威对照 |

---

*文档版本：单机部署方案，基于 06 实测环境（Lustre 2.17.0 + ZFS 2.2.2 + 6.8.0-138-generic）。命令中 IP/盘符/用户名为示例，执行前请按实际环境核对。*
*最后更新：2026-08-25 | 作者：WorkBuddy 文档团队*
