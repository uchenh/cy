# Lustre 单机部署照做版：192.168.12.160（Ubuntu 24.04.4 + Lustre 2.17.0 + ZFS）

> 本文是 `docs/10-lustre-单机虚拟机-部署方案.md` 在目标机器 **192.168.12.160** 上的**实例化照做版**——所有 IP、用户名、盘符均为该机实测值，命令可直接复制执行（需 sudo 密码时输入 `1`）。
>
> 机器环境已于 2026-08-26 通过 SSH 实测确认（见 §1），信息齐全，无需补充。

---

## 目录

0. [部署结果（实测）](#0-部署结果2026-08-26-实际执行完成)
1. [机器实测环境与盘符映射](#1-机器实测环境与盘符映射)
2. [部署前置要点](#2-部署前置要点)
3. [部署流程（8 步）](#3-部署流程8-步)
4. [部署成功判定清单](#4-部署成功判定清单)
5. [常见问题与排障](#5-常见问题与排障)
6. [参考信息](#6-参考信息)

---

## 0. 部署结果（2026-08-26 实际执行完成 ✅）

> 本文档流程已于 2026-08-26 在 192.168.12.160 上**实际部署成功**，以下为实测结果。

| 验证项 | 实测结果 |
|--------|----------|
| 内核 | 6.8.0-138-generic（GA，已 `apt-mark hold`；7.0 HWE 已 purge） |
| Lustre 服务端 | 2.17.0（b2_17 / commit 4b0407a，`make debs` 编译，7 个 .deb） |
| ZFS | 2.2.2-0ubuntu9.4 |
| LNet | `192.168.12.160@tcp`（ens33，状态 up） |
| 目标 | MGS / MDT0000 / OST0000 / OST0001 全部注册，`lctl dl` 24 项 UP |
| 客户端 | `192.168.12.160@tcp:/lustre` → `/mnt/lustre`（等待式 systemd 自启，active） |
| 容量 | MDT **18.7G** + OST0000 38.0G + OST0001 28.4G = **66.3G**（重建后已修复） |
| 条带化 | `lfs setstripe -c 2 -S 4M` 生效，文件横跨 OST0000/0001 |
| IO | `dd` 写 512M = **167 MB/s**（VM 本机回环） |
| 重启自愈 | 重启后 zpool 自动导入、目标自动挂载、客户端自动恢复（OST0000 需约 1 分钟 recovery 窗口后全部可见） |

**⚠️ 实测发现：磁盘设备名漂移（重要，已修复）**

部署前在 7.0 内核下 `lsblk` 探测为 sdb=20G / sdc=2G / sdd=40G / sde=30G；**切回 6.8 内核重启后设备名发生交换**（sdb↔sdc、sdd↔sde）。首轮建池因此错位（MGT 拿到 20G 盘、MDT 只有 2G 盘）。

**已于 2026-08-26 重建修复**：销毁四池后按**当前**设备名重建——`/dev/sdb`(2G)=MGT、`/dev/sdc`(20G)=MDT、`/dev/sde`(40G)=OST0、`/dev/sdd`(30G)=OST1；重建后 `lfs df` 正常显示 MDT 18.7G，重启自愈复测通过。当前池容量：

| 池 | 实际容量 | 物理盘 |
|----|----------|--------|
| lustre-mgt（MGS） | 1.88G | 2G 盘（sdb） |
| lustre-mdt（MDT） | 19.5G | 20G 盘（sdc） |
| lustre-ost0 | 39.5G | 40G 盘（sde） |
| lustre-ost1 | 29.5G | 30G 盘（sdd） |

> 🔧 教训：① 内核切换/重启后 `/dev/sdX` 设备名可能漂移，建池前务必**重新 `lsblk` 确认当前映射**；② 生产环境应使用 `/dev/disk/by-id/`（或 by-uuid/by-path）建池，规避漂移（对应 05 文档 §6.6 提示）；③ 池创建后 vdev 绑定设备内部 ID，后续重启不受设备名变化影响，仅"创建时"需注意。

---

## 1. 机器实测环境与盘符映射

**实测信息（2026-08-26 SSH 探查，用户 `ps`）**：

| 项 | 实测值 | 是否符合要求 |
|----|--------|--------------|
| 系统 | Ubuntu 24.04.4 LTS | ✅（与 10 文档一致） |
| 内核 | **7.0.0-30-generic（HWE 线）** | ⚠️ **必须按 §3.1 切回 6.8 GA**（Lustre 2.17 / ZFS 2.2.2 不支持 7.0） |
| CPU / 内存 | 4 vCPU / 7.7 GiB | ✅（满足推荐档） |
| 网卡 / IP | ens33 / 192.168.12.160/24 | ✅ |
| sudo | 用户 `ps` 在 sudo 组，需密码 `1` | ✅ |
| 代理 | 192.168.12.187:7897 端口开放 | ✅（编译 clone 走代理） |

**磁盘与 target 映射（实测 `lsblk`）**：

| 设备 | 容量 | 分配角色 | 依据 |
|------|------|----------|------|
| sda | 50G | 系统盘（ext4，/） | 不动 |
| **sdc** | **2G** | **MGT** | 2G 正好符合 MGT 1~2G 建议，小盘最合适 |
| **sdb** | **20G** | **MDT** | 符合 MDT 10~20G 建议 |
| **sdd** | **40G** | **OST0** | 数据盘 |
| **sde** | **30G** | **OST1** | 数据盘 |

> 池命名与 10/06 文档一致：`lustre-mgt` / `lustre-mdt` / `lustre-ost0` / `lustre-ost1`；OST index 从 0 开始。
> 注：机器还挂着 Ubuntu 安装 ISO（sr1），不影响部署，可忽略。

---

## 2. 部署前置要点

- 全程在 192.168.12.160 上执行，登录用户 `ps`，需提权命令用 `sudo`（密码 `1`）
- **内核是 7.0 HWE，必须先切 6.8 GA 再装 ZFS**（顺序不可反，否则 zfs-dkms 编译卡死）
- 编译 GitHub clone 必须走代理 `192.168.12.187:7897`
- 已确认 sdb/sdc/sdd/sde 四块数据盘均为裸盘（无分区/格式化/挂载），可直接建池

---

## 3. 部署流程（8 步）

### 3.1 内核修正：切回 6.8 GA 并锁定 `[192.168.12.160]`

```bash
# 1) 安装 GA 内核与头文件
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic
ls /boot/vmlinuz-*                  # 确认实际 6.8 版本号，记下来（06 实测为 6.8.0-138-generic）

# 2) 设置 GRUB 默认启动 6.8（菜单项用上一步 ls 输出的实际版本号）
#    示例基于 6.8.0-138-generic，若实际版本不同请替换版本号
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-138-generic"/' /etc/default/grub
grep ^GRUB_DEFAULT /etc/default/grub
sudo update-grub
sudo reboot

# 3) 重启后确认进入 6.8
uname -r                            # ✔ 期望 6.8.0-xxx-generic（GA 线）

# 4) 锁定内核 + 移除 HWE 7.0（顺序不可反）
sudo apt-mark hold linux-image-generic linux-headers-generic
sudo apt purge -y linux-image-7.0.0-30-generic linux-headers-7.0.0-30-generic \
  linux-image-generic-hwe-24.04 linux-headers-generic-hwe-24.04 \
  linux-modules-7.0.0-30-generic
```

> 🔧 若误装 zfs-dkms 导致 dpkg 卡死：`sudo pkill -f "dkms build -m zfs"` + `sudo dpkg --configure -a`。
> ⚠️ purge 的内核版本号（`7.0.0-30`）以 `uname -r` 实际输出为准。

### 3.2 基础环境 `[192.168.12.160]`

```bash
sudo hostnamectl set-hostname lustre-single
sudo apt install -y chrony curl wget
sudo systemctl enable --now chrony
sudo apt install -y git ca-certificates gnupg lsb-release software-properties-common \
  build-essential gcc make linux-headers-$(uname -r)
# 关闭 swap（防超时 evict）
sudo swapoff -a && sudo sed -i '/ swap /s/^/#/' /etc/fstab
```

### 3.3 ZFS 与编译依赖 `[192.168.12.160]`

```bash
sudo apt install -y build-essential gcc make flex bison pkg-config \
  zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
  libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
  python3-dev dpkg-dev \
  libzfslinux-dev module-assistant debhelper quilt
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs && echo ZFS_LOADED
zpool version                     # ✔ 期望 zfs-2.2.x
```

### 3.4 服务端源码编译 `[192.168.12.160]`（约 40 分钟，后台）

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

### 3.5 安装模块并配置 LNet `[192.168.12.160]`

```bash
cd ~/lustre-release/debs
sudo dpkg -i *.deb
sudo depmod -a
sudo modprobe libcfs && echo LIBCFS_OK
sudo modprobe lustre && echo LUSTRE_OK
lctl version                      # ✔ 期望 2.17.0

# LNet：网卡 ens33，本机 NID = 192.168.12.160@tcp
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lctl list_nids               # ✔ 期望 192.168.12.160@tcp
```

> 单机部署服务端 deb 已含客户端模块，无需另装 DKMS 客户端包。

### 3.6 建池 + 创建 MGS/MDT/OST `[192.168.12.160]`

```bash
# 1) 四个池（按 §1 映射：sdc=MGT、sdb=MDT、sdd=OST0、sde=OST1）
sudo zpool create lustre-mgt /dev/sdc
sudo zpool create lustre-mdt /dev/sdb
sudo zpool create lustre-ost0 /dev/sdd
sudo zpool create lustre-ost1 /dev/sde
sudo zpool list                   # ✔ 四个池在线

# 2) mkfs：--mgsnode 指向本机 NID
sudo mkfs.lustre --fsname=lustre --mgs --backfstype=zfs lustre-mgt/mgt
sudo mkfs.lustre --fsname=lustre --mdt --index=0 \
    --mgsnode=192.168.12.160@tcp --backfstype=zfs lustre-mdt/mdt
sudo mkfs.lustre --fsname=lustre --ost --index=0 \
    --mgsnode=192.168.12.160@tcp --backfstype=zfs lustre-ost0/ost
sudo mkfs.lustre --fsname=lustre --ost --index=1 \
    --mgsnode=192.168.12.160@tcp --backfstype=zfs lustre-ost1/ost

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

### 3.7 客户端挂载（本机即是客户端）`[192.168.12.160]`

```bash
sudo mkdir -p /mnt/lustre
sudo mount -t lustre 192.168.12.160@tcp:/lustre /mnt/lustre
mount | grep lustre               # ✔ 出现 192.168.12.160@tcp:/lustre on /mnt/lustre
lfs df -h /mnt/lustre             # ✔ MDT0000 + OST0000 + OST0001
```

### 3.8 开机自启 `[192.168.12.160]`

```bash
# 1) fstab：服务端目标 + ZFS cachefile
# 追加：
#   lustre-mgt/mgt /mnt/mgt lustre defaults,_netdev 0 0
#   lustre-mdt/mdt /mnt/mdt lustre defaults,_netdev 0 0
#   lustre-ost0/ost /mnt/ost0 lustre defaults,_netdev 0 0
#   lustre-ost1/ost /mnt/ost1 lustre defaults,_netdev 0 0
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-mgt lustre-mdt lustre-ost0 lustre-ost1

# 2) LNet 自启
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

# 3) 客户端挂载用等待式 systemd（防重启 recovery 窗口期丢挂载）
sudo tee /etc/systemd/system/lustre-client-mount.service >/dev/null <<'EOF'
[Unit]
Description=Mount Lustre client /mnt/lustre (waits for MGS)
After=network-online.target lnet-setup.service
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'for i in $(seq 1 30); do /usr/sbin/lnetctl ping 192.168.12.160@tcp >/dev/null 2>&1 && break; sleep 10; done'
ExecStart=/bin/mount -t lustre 192.168.12.160@tcp:/lustre /mnt/lustre

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now lustre-client-mount
```

---

## 4. 部署成功判定清单

> 全部 ✔ 即部署成功。判定后建议在客户端做一次功能验证：

```bash
sudo mkdir -p /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test
sudo dd if=/dev/zero of=/mnt/lustre/test/bigfile bs=1M count=512 oflag=direct conv=fdatasync
lfs getstripe -v /mnt/lustre/test/bigfile   # ✔ obdidx 含 0 和 1
```

| # | 验证项 | 命令 | ✔ 标准 |
|---|--------|------|--------|
| 1 | 内核正确 | `uname -r` | 6.8.0-xxx-generic（GA，已 hold） |
| 2 | 服务端版本 | `lctl version` | 2.17.0 |
| 3 | ZFS 加载 | `zpool version` | zfs-2.2.x |
| 4 | LNet 就绪 | `sudo lctl list_nids` | 192.168.12.160@tcp |
| 5 | 目标注册 | `sudo lctl get_param mgs.MGS.live.*` | MDT0000 + OST0000 + OST0001 |
| 6 | 客户端挂载 | `mount \| grep lustre` | 192.168.12.160@tcp:/lustre on /mnt/lustre |
| 7 | 容量正确 | `lfs df -h /mnt/lustre` | MDT(20G) + OST0000(40G) + OST0001(30G) |
| 8 | 条带化生效 | `lfs getstripe -v` | obdidx 含 0 和 1 |
| 9 | IO 可写 | `dd` 512M | 无报错 |
| 10 | 重启自愈 | `sudo reboot` 后 `lfs df -h` | 自动恢复 |

---

## 5. 常见问题与排障

| # | 现象 | 根因 | 解决 |
|---|------|------|------|
| 1 | `./configure --with-zfs` 报 missing zfs development headers | 缺 ZFS 开发头文件 | `sudo apt install libzfslinux-dev` |
| 2 | `make debs` 报 Unmet build dependencies | 缺 module-assistant/debhelper/quilt | `sudo apt install module-assistant debhelper quilt` |
| 3 | `zfs-dkms` 安装卡死 | 在 7.0 HWE 内核上构建 ZFS 失败 | 先切 6.8 GA 并 purge 7.0 再装（§3.1） |
| 4 | GitHub clone 极慢/卡住 | 直连限速 | 走代理 `192.168.12.187:7897`（§3.4） |
| 5 | GRUB 切换后仍进 7.0 | 菜单项版本号与 `ls /boot/vmlinuz-*` 实际不符 | 用实际版本号重写 `GRUB_DEFAULT`，`update-grub` 后重启 |
| 6 | 重启后挂载丢失 / EBUSY | recovery 窗口期内客户端提前挂载 | 用 §3.8 等待式 systemd；`sudo lctl get_param mdt.lustre-MDT0000.recovery_status` 确认 COMPLETE |
| 7 | `lfs df` 只显示部分 OST | 有 OST 未挂载 | `mount \| grep lustre`、`lctl dl` 核对 |
| 8 | 编译 OOM / IO 卡顿 | 内存 7.7G 偏紧 | 关闭 GUI/后台程序；swap 已关确认 |

---

## 6. 参考信息

| 资料 | 位置 / 地址 | 用途 |
|------|-------------|------|
| 本文（11） | `docs/11-lustre-单机192.168.12.160-部署方案.md` | 本机照做版（本文） |
| 10 VM 单机方案 | `docs/10-lustre-单机虚拟机-部署方案.md` | 通用流程与 VM 排障 |
| 09 物理机单机方案 | `docs/09-lustre-单机测试机-部署方案.md` | 生产实际配置与局限 |
| 06 部署记录 | `docs/06-lustre-ubuntu24.04-3节点-部署方案.md` | 命令基准与踩坑记录 |
| 05 部署指南 | `docs/05-lustre-ubuntu24.04-部署指南.md` | 版本选型与排障方法论 |
| Lustre 官方 Wiki | https://wiki.lustre.org/ | Quick Start / Building / 运维手册 |

---

*文档版本：192.168.12.160 实例化照做版，环境实测于 2026-08-26。命令中 GRUB 版本号以 `ls /boot/vmlinuz-*` 实际输出为准。*
*最后更新：2026-08-26 | 作者：WorkBuddy 文档团队*
