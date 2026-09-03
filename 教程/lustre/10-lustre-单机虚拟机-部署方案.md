# Lustre 单机部署验证：单台虚拟机部署方案（Ubuntu 24.04 + Lustre 2.17.0 + ZFS）

> 面向"在一台虚拟机上先测试 Lustre 单机（All-in-One）能不能部署"的场景。**结论先行：可以，且这是投入最小、最安全的验证方式**——在 VM 上跑通全流程（编译 → 建池 → MGS/MDT/OST → 挂载 → 条带化 → 重启自愈），确认单机形态可行后，再决定是否在借来的物理测试机上按 `docs/09` 部署。
>
> 本文命令与 09 文档（物理机单机方案）**完全同构**，差异仅在 VM 配置档位（实验验证用）与"部署成功判定清单"；与 06 实测环境（Ubuntu 24.04 + Lustre 2.17.0 + ZFS 2.2.2 + 6.8.0-138-generic）严格对齐。

---

## 目录

1. [结论先行：VM 单机部署验证方案](#1-结论先行为什么先在-vm-验证)
2. [VM 配置建议（实验验证档）](#2-vm-配置建议实验验证档)
3. [部署流程（全在一台 VM）](#3-部署流程全在一台-vm)
4. [部署成功判定清单](#4-部署成功判定清单)
5. [常见问题与排障（含 VM 特有）](#5-常见问题与排障含-vm-特有)
6. [参考信息](#6-参考信息)

---

## 1. 结论先行：为什么先在 VM 验证

| 维度 | 说明 |
|------|------|
| 能否单机部署 | **能**。Lustre 组件是逻辑角色而非物理要求，MGS/MDS/OSS/Client 可全部跑在同一台（官方 Quick Start 支持 All-in-One） |
| 为什么先 VM | 验证"能不能部署"只需功能跑通，VM 足够；编译、建池、挂载、条带化功能与物理机完全一致；失败不污染物理环境 |
| VM 验证的边界 | 性能数字无参考价值（VM 磁盘/网络虚拟化），HA/快照等操作被 04 §3.3 明确禁止；验证目的是**流程可行性**，不是性能 |
| 验证通过后 | 按 `docs/09-lustre-单机测试机-部署方案.md` 在物理测试机上以生产实际配置重跑一遍即可（命令几乎相同） |

**适用**：学习验证、跑通全流程、给同事演示、为物理机部署踩坑排雷。
**不适用**：性能基准、HA 演练（单机形态本来就不支持）。

---

## 2. VM 配置建议（实验验证档）

> 依据 `docs/04` §3.1（单节点最小可跑）与 §3.3（磁盘/控制器）。目标是**跑通功能**，不需要生产规格；但内存别太抠，ZFS ARC 默认吃一半内存。

| 项 | 建议 | 说明 |
|----|------|------|
| 平台 | VMware Workstation / KVM / VirtualBox | 与 06 环境一致用 VMware 亦可 |
| CPU | **4 vCPU**（2 vCPU 也能跑） | 编译 `-j4` 更快；04 单节点最小 2 vCPU |
| 内存 | **8 GB**（最小 4 GB） | ZFS ARC 占一半；4 GB 编译+运行偏紧，8 GB 从容 |
| 系统盘 | **50 GB**（thin 可，介意性能用 thick） | 系统 + 编译产物（源码约 128M + debs） |
| 数据盘 | **MGT 1~2 GB + MDT 10~20 GB + OST 20 GB×2**，一 target 一块独立虚拟盘 | 每类 target 独立裸盘，不要和系统盘混用 |
| 网卡 | 1 块（VMware: e1000/vmxnet3；KVM: virtio；VBox: 默认） | LNet 走 `@tcp`，端口 988 |
| 磁盘控制器 | VMware 用 **PVSCSI**；KVM 用 **virtio-scsi**；VBox 用 SATA | 性能与热插拔支持更好（04 §3.3） |

**VM 创建注意点**（依据 04 §3.3 / §3.5）：

- 一 target 一块虚拟盘：MGT、MDT、OST0、OST1 各对应一块裸盘（如 `/dev/sdb`~`/dev/sde`），不要在系统盘上为多个 target 建池
- ⚠️ **绝对不要对运行中的 Lustre 节点做整机快照/回滚**——MGS 配置日志与目标磁盘代次必须严格一致，回滚会导致整机无法挂载（04 §3.3）
- 安装 Ubuntu 24.04 LTS 后，若内核是 7.0 HWE 线，按 §3.1 切回 6.8 GA 并锁定

---

## 3. 部署流程（全在一台 VM）

> 与 09 §4 完全同构，命令可直接照做。假设：登录用户 `user`，网卡 `ens33`，数据盘 `/dev/sdb`(MGT)、`/dev/sdc`(MDT)、`/dev/sdd`、`/dev/sde`(OST×2)，本机 IP 示例 `192.168.12.230`。按实际替换。

### 3.1 环境探查与内核修正 `[VM]`

```bash
cat /etc/os-release | head -3
uname -r                          # 若为 7.0.0-xx-generic（HWE 线），执行下方内核修正
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE   # ✔ 确认 4 块数据盘为裸盘

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

### 3.2 基础环境 `[VM]`

```bash
sudo hostnamectl set-hostname lustre-vm
sudo apt install -y chrony
sudo systemctl enable --now chrony
sudo apt install -y git curl wget ca-certificates gnupg lsb-release software-properties-common \
  build-essential gcc make linux-headers-$(uname -r)
# 关闭/调低 swap（防超时 evict，见 04 §3.5）
sudo swapoff -a && sudo sed -i '/ swap /s/^/#/' /etc/fstab
```

### 3.3 ZFS 与编译依赖 `[VM]`

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

### 3.4 服务端源码编译 `[VM]`（约 40 分钟，后台）

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

### 3.5 安装模块并配置 LNet `[VM]`

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
sudo lctl list_nids               # ✔ 期望 192.168.12.230@tcp
```

> 单机部署服务端 deb 已包含客户端功能（llite/osc 模块），无需再装 Azure DKMS 包。

### 3.6 建池 + 创建 MGS/MDT/OST `[VM]`

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
    --mgsnode=192.168.12.230@tcp --backfstype=zfs lustre-mdt/mdt
sudo mkfs.lustre --fsname=lustre --ost --index=0 \
    --mgsnode=192.168.12.230@tcp --backfstype=zfs lustre-ost0/ost
sudo mkfs.lustre --fsname=lustre --ost --index=1 \
    --mgsnode=192.168.12.230@tcp --backfstype=zfs lustre-ost1/ost

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

### 3.7 客户端挂载（本机即是客户端）`[VM]`

```bash
sudo mkdir -p /mnt/lustre
sudo mount -t lustre 192.168.12.230@tcp:/lustre /mnt/lustre
mount | grep lustre               # ✔ 出现 192.168.12.230@tcp:/lustre on /mnt/lustre
lfs df -h /mnt/lustre             # ✔ MDT0000 + OST0000 + OST0001
```

### 3.8 开机自启 `[VM]`

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
ExecStart=/bin/bash -c 'for i in $(seq 1 30); do /usr/sbin/lnetctl ping 192.168.12.230@tcp >/dev/null 2>&1 && break; sleep 10; done'
ExecStart=/bin/mount -t lustre 192.168.12.230@tcp:/lustre /mnt/lustre

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now lustre-client-mount
```

---

## 4. 部署成功判定清单

> 逐项勾选，全部 ✔ 即"单机部署验证通过"。这也是后续物理机部署的验收标准（命令相同）。

| # | 验证项 | 命令 | ✔ 标准 |
|---|--------|------|--------|
| 1 | 内核正确 | `uname -r` | 6.8.0-138-generic（GA，已 hold） |
| 2 | 服务端版本 | `lctl version` | 2.17.0 |
| 3 | ZFS 加载 | `zpool version` | zfs-2.2.2-0ubuntu9.4 |
| 4 | LNet 就绪 | `sudo lctl list_nids` | 本机 IP@tcp |
| 5 | 目标全部注册 | `sudo lctl get_param mgs.MGS.live.*` | MDT0000 + OST0000 + OST0001 全部列出 |
| 6 | 客户端挂载 | `mount \| grep lustre` | 192.168.12.230@tcp:/lustre on /mnt/lustre |
| 7 | 容量正确 | `lfs df -h /mnt/lustre` | MDT + 2×OST，无异常 0 值 |
| 8 | 条带化生效 | `lfs getstripe -v /mnt/lustre/test/bigfile` | obdidx 含 0 和 1 |
| 9 | IO 可写 | `dd if=/dev/zero of=/mnt/lustre/test/bigfile bs=1M count=512 oflag=direct conv=fdatasync` | 无报错，字节数正确 |
| 10 | 重启自愈 | `sudo reboot` 后 `lfs df -h` | 自动恢复，客户端服务 active |

**功能验证（第 8 项前置）**：

```bash
sudo mkdir -p /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test
sudo dd if=/dev/zero of=/mnt/lustre/test/bigfile bs=1M count=512 oflag=direct conv=fdatasync
lfs getstripe -v /mnt/lustre/test/bigfile
```

> 性能参考（不具代表性）：06 实测 3 节点 1GbE 下 dd 约 114 MB/s；VM 单机受虚拟磁盘/内存影响，数字仅供"能跑"确认。

---

## 5. 常见问题与排障（含 VM 特有）

| # | 现象 | 根因 | 解决 |
|---|------|------|------|
| 1 | `./configure --with-zfs` 报 missing zfs development headers | 缺 ZFS 开发头文件 | `sudo apt install libzfslinux-dev` |
| 2 | `make debs` 报 Unmet build dependencies | 缺 module-assistant/debhelper/quilt | `sudo apt install module-assistant debhelper quilt` |
| 3 | `zfs-dkms` 安装卡死、dpkg 锁被占 | 在 7.0 HWE 内核上构建 ZFS 失败 | 先切 6.8 GA 并 purge 7.0 再装（§3.1）；卡死时 `pkill -f "dkms build -m zfs"` + `dpkg --configure -a` |
| 4 | GitHub clone 极慢/卡住 | 直连限速 | 走代理 `192.168.12.187:7897` |
| 5 | 本机 mount 报 EBUSY / 连不上 MGS | 重启后 MGS recovery 窗口期内客户端提前挂载 | 用 §3.8 等待式 systemd；`sudo lctl get_param mdt.lustre-MDT0000.recovery_status` 确认 COMPLETE |
| 6 | `lfs df` 只显示部分 OST | 有 OST 未挂载 | `mount \| grep lustre`、`lctl dl` 核对目标状态 |
| 7 | 编译/OOM 或 IO 卡顿 | VM 内存不足（4 GB 偏紧） | 调高 VM 内存至 8 GB；确认 swap 已关 |
| 8 | dd 首次写满突然变慢 | thin 虚拟盘首次分配块 | 介意则换 thick 盘（04 §3.3） |
| 9 | 整机快照回滚后无法挂载 | MGS 配置日志与目标代次失配 | **不要对运行中的 Lustre VM 做快照/回滚**（04 §3.3） |
| 10 | 客户端被 evict | 时钟漂移 / 超时 | chrony 校时；VM 挂起/恢复易触发，避免频繁挂起 |

---

## 6. 参考信息

| 资料 | 位置 / 地址 | 用途 |
|------|-------------|------|
| 本文（10） | `docs/10-lustre-单机虚拟机-部署方案.md` | VM 单机部署验证（本文） |
| 09 物理机单机方案 | `docs/09-lustre-单机测试机-部署方案.md` | 验证通过后在物理测试机部署（生产实际配置） |
| 06 部署记录 | `docs/06-lustre-ubuntu24.04-3节点-部署方案.md` | 3 节点 VM 全流程（命令基准） |
| 05 部署指南 | `docs/05-lustre-ubuntu24.04-部署指南.md` | 版本选型、后端选择、排障方法论 |
| 04 VM 指南 | `docs/04-lustre-虚拟机部署指南.md` | §3 VM 配置建议、磁盘/网络/虚拟化注意点 |
| 08 扩容指南 | `docs/08-lustre-新增OSS节点与数据盘.md` | 若后续并入现有集群当 OSS 节点 |
| Lustre 官方 Wiki | https://wiki.lustre.org/ | Quick Start / Building / 运维手册 |
| Whamcloud Support Matrix | https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix | 版本 × 发行版 × 内核 × ZFS 权威对照 |

---

*文档版本：VM 单机部署验证方案，基于 06 实测环境（Lustre 2.17.0 + ZFS 2.2.2 + 6.8.0-138-generic）。命令中 IP/盘符/用户名为示例，执行前请按实际环境核对。*
*最后更新：2026-08-26 | 作者：WorkBuddy 文档团队*
