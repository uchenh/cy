# Lustre 3 节点集群实际部署操作记录（Ubuntu 24.04 + Lustre 2.17.0 + ZFS）

> **本文档记录 2026-08-20 在三台 Ubuntu 24.04 虚拟机上真实完成的一次 Lustre 2.17.0 全栈部署**：从环境探查、内核修正、服务端源码编译，到目标创建、客户端挂载、开机自启与重启自愈验证的**全部实际操作过程与结果**。所有 IP/用户名/设备名均来自实际环境，命令为当时真实执行（含 `sudo` 用法），可直接复现。
>
> 补充说明：本文为「实际部署操作记录」，不含规划方案内容；方法论与原理性细节见同目录 `docs/05-lustre-ubuntu24.04-部署指南.md`。

---

## 1. 部署结果摘要（结论先行）

- **集群构成**：3 台 Ubuntu 24.04.2 LTS 虚拟机，Lustre **2.17.0**（服务端源码编译 + 客户端 DKMS），存储后端 **ZFS 2.2.2**，网络 LNet **tcp**（1GbE）。
- **挂载**：客户端 `client1` 上 `192.168.12.220@tcp:/lustre` → `/mnt/lustre`。
- **容量**：MDT0000 23.5G + OST0000/OST0001 各 47.6G = **合计 95.2G**。
- **验证通过**：三机 LNet 互 ping 通；`lfs setstripe -c 2` 文件确认横跨 OST0000/OST0001 两个 OST；`dd` 写 512M 约 **114 MB/s**（1GbE 带宽上限附近）。
- **开机自愈**：三台重启后 ZFS 池自动导入、Lustre 目标自动挂载、客户端自动挂载恢复，无需人工干预。
- **关键前置修正**：三台初始内核为 7.0.0-29-generic（HWE 线），与 Lustre 2.17 / ZFS 2.2.2 支持的内核线不符，**已切回 6.8.0-138-generic（GA）并锁定**（详见 §3.2）。

| 节点 | 主机名 | IP | 登录用户 | 角色 |
| --- | --- | --- | --- | --- |
| mds1 | mds1 | 192.168.12.220 | user | MGS + MDS（MDT0000） |
| oss1 | oss1 | 192.168.12.221 | user2 | OSS（OST0000 / OST0001） |
| client1 | client1 | 192.168.12.222 | user1 | Client |

> 三个用户 sudo 密码均为 `1`；所有需提权的命令在文中均带 `sudo`。

---

## 2. 实际环境信息

### 2.1 硬件与磁盘

| 节点 | vCPU | 内存 | 系统盘 | 数据盘（实际设备/容量/用途） |
| --- | --- | --- | --- | --- |
| mds1 | 4 | 3.8G | sda 50G（ext4，/） | sdb 5G = **MGT**；sdc 25G = **MDT** |
| oss1 | 4 | 3.8G | sda 50G（ext4，/） | sdb 50G = **OST0**；sdc 50G = **OST1** |
| client1 | 4 | 3.8G | sda 40G（ext4，/） | 无 |

- 数据盘均为**裸盘**（未分区、未格式化、未挂载），直接交给 ZFS 建池。
- 虚拟化平台：VMware；网卡 **ens33**；静态 IP，网关 192.168.12.254；时区 Asia/Shanghai。

### 2.2 软件版本（实际安装）

| 组件 | 版本 |
| --- | --- |
| 发行版 | Ubuntu 24.04.2 LTS（noble） |
| 内核（最终） | **6.8.0-138-generic**（GA，`apt-mark hold` 锁定） |
| Lustre 服务端 | **2.17.0**（git `b2_17` 分支，commit `4b0407a New release 2.17.0`，`make debs` 编译） |
| Lustre 客户端 | **2.17.0**（Azure DKMS 包 `amlfs-lustre-client-dkms-2.17.0-24-gf517bc4`） |
| ZFS | 2.2.2-0ubuntu9.4（含 zfsutils-linux、zfs-dkms、libzfslinux-dev） |
| gcc | 13（Ubuntu 24.04 默认） |

---

## 3. 部署操作全过程

> 以下步骤按实际执行顺序记录。`<节点>` 标注该步在哪台机器上执行；"三台"= mds1/oss1/client1。

### 3.1 环境探查（部署前）

```bash
# 三台分别确认 OS / 内核 / 磁盘 / sudo 可用性（mds1 示例；oss1/client1 类似）
ssh user@192.168.12.220        # 或按实际登录方式
cat /etc/os-release | head -3
uname -r                       # 实测为 7.0.0-29-generic（HWE，见 §3.2）
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
id; sudo -n true && echo NOPASS || echo NEEDS_PASS   # 实测 sudo 需密码
```

**探查结果要点**：
- 三台均为 Ubuntu 24.04 LTS，但内核是 **7.0.0-29-generic（HWE 线）**，不是 6.8 GA；
- 磁盘布局与 §2.1 一致，数据盘为裸盘；
- sudo 需要密码（密码 `1`）。

### 3.2 内核修正：切回 6.8 GA 并锁定（关键前置，三台执行）

> **为什么必须做**：Lustre 2.17 官方支持 Ubuntu 24.04 的 **6.8 GA 内核线**；且 ZFS 2.2.2 官方仅兼容内核 3.10~6.6，在 7.0 内核上 `zfs-dkms` 编译必然失败。实测若不先切内核，后续 ZFS/Lustre 均无法构建。

```bash
# 1) 安装 GA 内核与头文件（6.8 线）
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic
ls /boot/vmlinuz-*              # 确认出现 vmlinuz-6.8.0-138-generic

# 2) 设置 GRUB 默认启动 6.8（Ubuntu 24.04 实测菜单项写法）
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-138-generic"/' /etc/default/grub
grep ^GRUB_DEFAULT /etc/default/grub
sudo update-grub

# 3) 重启并确认进入 6.8
sudo reboot
uname -r                        # 期望 6.8.0-138-generic

# 4) 锁定内核，防止 apt upgrade 换内核导致模块失效
sudo apt-mark hold linux-image-generic linux-headers-generic

# 5) 移除 HWE 7.0 内核（实测必须，否则 zfs-dkms 会为 7.0 构建失败卡死 dpkg）
sudo apt purge -y linux-image-7.0.0-29-generic linux-headers-7.0.0-29-generic \
  linux-image-generic-hwe-24.04 linux-headers-generic-hwe-24.04 \
  linux-modules-7.0.0-29-generic
```

> ⚠️ **实测踩坑**：若先装 `zfs-dkms` 再 purge 7.0，安装会卡在「为 7.0 内核构建 ZFS 模块」（ZFS 2.2.2 不支持 7.0），dpkg 锁被长期占用。正确顺序是**先切 6.8 并 purge 7.0，再装 ZFS**。修复命令：`sudo pkill -f "dkms build -m zfs"` + `sudo dpkg --configure -a`。

### 3.3 基础环境准备（三台执行）

```bash
# 1) 主机名（mds1/oss1/client1 各自设置）
sudo hostnamectl set-hostname mds1

# 2) /etc/hosts 三机映射（三台内容一致；实测需删除默认的 127.0.1.1 行避免歧义）
sudo tee -a /etc/hosts <<'EOF'
192.168.12.220  mds1
192.168.12.221  oss1
192.168.12.222  client1
EOF
sudo sed -i '/^127.0.1.1 /d' /etc/hosts
getent hosts mds1 oss1 client1

# 3) 时间同步（Lustre 对时钟敏感，防客户端被 evict）
sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc sources -v

# 4) 公共工具 + 与运行内核严格一致的头文件
sudo apt install -y git curl wget ca-certificates gnupg lsb-release software-properties-common \
  build-essential gcc make linux-headers-$(uname -r)
```

> **实测差异**：实验网络隔离，ufw 保持未启用（LNet 988 端口不受阻）；若启用防火墙需 `sudo ufw allow 988/tcp`。

### 3.4 服务端依赖与 ZFS（mds1、oss1 执行）

```bash
# 1) 编译依赖（实测需补 libzfslinux-dev / module-assistant / debhelper / quilt）
sudo apt install -y build-essential gcc make flex bison pkg-config \
  zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
  libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
  python3-dev dpkg-dev \
  libzfslinux-dev module-assistant debhelper quilt

# 2) ZFS（zfs-dkms 会按当前 6.8 内核编译模块）
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs && echo ZFS_LOADED
zpool version          # 期望 zfs-2.2.2-0ubuntu9.4
ls /usr/src | grep zfs # 期望 /usr/src/zfs-2.2.2
```

> **实测差异（相对原 06 方案）**：configure 需要 ZFS 开发头文件（`libzfslinux-dev`）；`make debs` 需要 `module-assistant debhelper quilt`。缺任一都会失败。

### 3.5 服务端源码编译（mds1、oss1 执行）

> GitHub 直连在本网络很慢，**必须走代理** `192.168.12.187:7897`（本机代理）。源码约 128M。

```bash
# 1) 克隆并检出 b2_17 分支（Lustre 2.17 稳定线）
export http_proxy=http://192.168.12.187:7897 https_proxy=http://192.168.12.187:7897
git clone --depth 1 https://github.com/lustre/lustre-release.git
cd ~/lustre-release
git fetch --depth 1 origin b2_17
git checkout -b b2_17 FETCH_HEAD
git log --oneline -1     # 期望 4b0407a New release 2.17.0

# 2) 生成 configure 并配置（ZFS 后端、禁用 ldiskfs）
sh autogen.sh
./configure --enable-server --with-zfs --disable-ldiskfs \
    --with-linux=/usr/src/linux-headers-$(uname -r)
# 成功标志：结尾 "Type 'make' to build Lustre." + "checking if ZFS has ... yes"

# 3) 编译打包（实测约 40 分钟，4 vCPU/3.8G 用 -j4）
nohup make debs -j4 > /tmp/lustre_make.log 2>&1 &
# 轮询：tail /tmp/lustre_make.log；产出 7 个 .deb（含 lustre-server-modules-6.8.0-138-generic）
ls ~/lustre-release/debs/*.deb
```

> **实测说明**：`osd-zfs` 内核模块（osd_zfs.ko）**内含于 `lustre-server-modules-6.8.0-138-generic` 包**，没有独立的 osd-zfs .deb，属正常。

### 3.6 安装服务端 .deb 并加载模块（mds1、oss1 执行）

```bash
cd ~/lustre-release/debs
sudo dpkg -i *.deb
sudo depmod -a
sudo modprobe libcfs && echo LIBCFS_OK
sudo modprobe lustre && echo LUSTRE_OK
lctl version               # 期望 2.17.0
lsmod | grep -E 'lustre|osd_zfs|obdfilter' | head
```

### 3.7 LNet 配置（三台执行）

```bash
# 本机网卡为 ens33；2.17 下 NID 显示为 192.168.12.x@tcp
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lnetctl net show
sudo lctl list_nids        # 期望 192.168.12.220@tcp（mds1）等
```

连通性验证（三机互 ping）：

```bash
# mds1 上 ping oss1 / client1
sudo lnetctl ping 192.168.12.221@tcp
sudo lnetctl ping 192.168.12.222@tcp
# client1 上 ping mds1 / oss1（客户端必须能到 MGS）
sudo lnetctl ping 192.168.12.220@tcp
sudo lnetctl ping 192.168.12.221@tcp
```

> **实测差异**：方案示例写 `@tcp0`，2.17 实际 `lctl list_nids` 显示 `@tcp`，二者对应同一网络，互 ping 用 `@tcp` 即可。

### 3.8 创建并挂载 MGS / MDT（mds1 执行）

```bash
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
sudo mount -t lustre lustre-mgt/mgt /mnt/mgt
sudo mount -t lustre lustre-mdt/mdt /mnt/mdt
mount | grep lustre
```

### 3.9 创建并挂载 OST（oss1 执行）

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
```

### 3.10 服务端目标验证（mds1 执行）

```bash
sudo lctl dl                      # 期望 MGS / MDT0000 / 相关 OSC 行均 UP
sudo lctl get_param mgs.MGS.live.*   # 期望列出 lustre-MDT0000 / OST0000 / OST0001
```

**实测输出摘要**（mds1 `lctl dl`）：`MGS`、`lustre-MDT0000` 等 12 行 UP；`mgs.MGS.live.lustre` 下可见 `lustre-MDT0000`、`lustre-OST0000`、`lustre-OST0001` 三项——**全部目标注册成功**。

### 3.11 客户端安装与挂载（client1 执行）

```bash
# 1) 配置 Azure Managed Lustre 仓库（DKMS 路径）
sudo apt install -y ca-certificates curl apt-transport-https lsb-release gnupg dpkg-dev
source /etc/lsb-release
ARCH=$(dpkg-architecture -q DEB_BUILD_ARCH)
echo "deb [arch=${ARCH}] https://packages.microsoft.com/repos/amlfs-${DISTRIB_CODENAME}/ ${DISTRIB_CODENAME} main" \
  | sudo tee /etc/apt/sources.list.d/amlfs.list
curl -sL https://packages.microsoft.com/keys/microsoft.asc \
  | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
sudo apt update

# 2) 查询并安装 DKMS 包（实测包名 amlfs-lustre-client-dkms-2.17.0-24-gf517bc4）
apt-cache search amlfs-lustre-client-dkms
sudo apt install -y amlfs-lustre-client-dkms-2.17.0-24-gf517bc4
# DKMS 按当前内核（6.8.0-138）自动编译模块

# 3) 加载模块并验证（必须 sudo）
sudo modprobe lustre
lctl version                # 期望 2.17.0_24_gf517bc4
lsmod | grep -E 'lustre|mdc|lov'

# 4) LNet（同 §3.7）+ 挂载
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lnetctl ping 192.168.12.220@tcp    # 先确认 MGS 可达
sudo mkdir -p /mnt/lustre
sudo mount -t lustre 192.168.12.220@tcp:/lustre /mnt/lustre
mount | grep lustre
```

> **实测差异**：普通用户 `modprobe lustre` 报 `Operation not permitted`，**必须 sudo**。

### 3.12 功能验证（client1 执行）

```bash
# 1) 容量总览：应看到 MDT0000 + OST0000 + OST0001
lfs df -h /mnt/lustre

# 2) 条带化：目录设默认布局跨 2 OST，创建文件验证分布
sudo mkdir -p /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test/striped2
lfs getstripe -v /mnt/lustre/test/striped2   # lmm_stripe_count: 2；obdidx 含 0 和 1

# 3) IO 测试
dd if=/dev/zero of=/mnt/lustre/test/bigfile bs=1M count=512 oflag=direct conv=fdatasync
```

**实测结果**：
- `lfs df -h`：MDT0000 23.5G、OST0000 47.6G、OST0001 47.6G，**filesystem_summary 95.2G**；
- `lfs getstripe -v striped2`：`lmm_stripe_count: 2`，对象落在 **obdidx 0 与 1**（OST0000 与 OST0001）——条带化生效；
- `dd` 写 512M：**536870912 bytes copied, 4.69s, 114 MB/s**（1GbE 上限附近）。

### 3.13 开机自启配置（三台分别执行）

```bash
# 1) fstab：服务端目标与客户端挂载均需 _netdev
# mds1 追加：
#   lustre-mgt/mgt /mnt/mgt lustre defaults,_netdev 0 0
#   lustre-mdt/mdt /mnt/mdt lustre defaults,_netdev 0 0
# oss1 追加：
#   lustre-ost0/ost /mnt/ost0 lustre defaults,_netdev 0 0
#   lustre-ost1/ost /mnt/ost1 lustre defaults,_netdev 0 0
# client1 追加：
#   192.168.12.220@tcp:/lustre /mnt/lustre lustre defaults,_netdev 0 0

# 2) ZFS 池写入 cachefile（Ubuntu 默认即 /etc/zfs/zpool.cache，保险起见显式设置）
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-mgt lustre-mdt   # mds1
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-ost0 lustre-ost1 # oss1

# 3) LNet 开机自启（三台，--if ens33）
sudo tee /etc/systemd/system/lnet-setup.service >/dev/null <<'EOF'
[Unit]
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
```

### 3.14 重启自愈验证（按 mds1 → oss1 → client1 顺序）

```bash
sudo reboot   # 依次重启三台，每台等约 75~90 秒后重连验证
```

**重启后实测状态**：

| 节点 | 内核 | ZFS 池 | Lustre 目标 | 挂载 |
| --- | --- | --- | --- | --- |
| mds1 | 6.8.0-138-generic | lustre-mgt / lustre-mdt 自动导入 | **12 个 UP**（MGS/MDT0000 等） | /mnt/mgt、/mnt/mdt 自动挂载 |
| oss1 | 6.8.0-138-generic | lustre-ost0 / lustre-ost1 自动导入 | **8 个 UP**（OST0000/0001 等） | /mnt/ost0、/mnt/ost1 自动挂载 |
| client1 | 6.8.0-138-generic | — | 6 个客户端目标 | /mnt/lustre 自动挂载，`lfs df -h` 正常 |

> **遗留提示**：`lnet-setup.service` 在部分节点重启后显示 `failed`，但 LNet 已随挂载自动 up（`lctl list_nids` 正常）。如需严格自启，可在 unit 中追加 `ExecStartPre=/usr/sbin/modprobe lnet`。

### 3.15 故障记录：三台同时重启后 client1 的 /mnt/lustre 未挂载（已解决）

> 现象：三台虚拟机**同时**重启后，client1 上 `mount | grep lustre` 无输出、`ls /mnt/lustre` 报目录不存在，看起来"挂载的盘没了"；mds1/oss1 服务端目标均正常。

**根因（两层）**：

1. **MGS/MDT 的 recovery 窗口期**：mds1 重启后 MDT 进入 `RECOVERING` 状态（默认最长 300s，实测 `recovery_duration: 300`），此期间拒绝客户端连接（内核日志 `mds_connect to node 192.168.12.220@tcp failed: rc = -16`，即 EBUSY）。
2. **fstab `_netdev` 只等网卡、不等服务**：客户端开机早期即尝试挂载 → 被 recovery 窗口拒绝 → systemd 挂载单元超时（默认 90s；设 `x-systemd.mount-timeout=300` 后仍因窗口未结束而失败）→ **失败后不再自动重试**，表现为"盘没了"。

**排查命令**（按序确认）：

```bash
# client1：确认服务端是否可达、目标是否注册
sudo lnetctl ping 192.168.12.220@tcp      # 通 → 网络层 OK
sudo lctl dl | grep -cE ' UP '            # 客户端设备栈应 UP
mount | grep lustre                        # 应为空（未挂载）
# mds1：确认 MDT recovery 状态
sudo lctl get_param mdt.lustre-MDT0000.recovery_status
# 期望 status: COMPLETE（RECOVERING 期间客户端连不上）
# client1 内核日志定位 rc=-16 来源
sudo journalctl -k | grep -i lustre | tail
```

**解决方案（已实施并验证）**：不用 fstab 挂载客户端，改为自定义 systemd 服务——**先循环 ping MGS 直到可达（最多 300s），再执行 mount**，服务挂载成功后保持 active：

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
ExecStart=/bin/bash -c 'for i in $(seq 1 30); do /usr/sbin/lnetctl ping 192.168.12.220@tcp >/dev/null 2>&1 && break; sleep 10; done'
ExecStart=/bin/mount -t lustre 192.168.12.220@tcp:/lustre /mnt/lustre
ExecStartPost=/bin/sleep 2

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl enable --now lustre-client-mount
sudo systemctl is-active lustre-client-mount   # 期望 active
lfs df -h /mnt/lustre                          # 期望 MDT + 2×OST
```

**验证结果**：三台同时重启后，mds1 目标 12 个 UP、oss1 8 个 UP、client1 的 `lustre-client-mount` 服务等待 MGS recovery 完成后自动挂载成功，`lfs df -h` 正常显示 MDT0000 + OST0000/0001。

> **经验总结**：Lustre 客户端挂载不建议依赖 fstab 的 `_netdev`（它不等服务端 recovery），生产环境应使用「等待 MGS 可达再挂载」的 systemd 服务（或 `x-systemd.automount` 懒挂载 + 说明文档），避免三台同时断电重启后客户端"盘丢失"的误判。

---

## 4. 最终验证结果汇总

| 验证项 | 命令 | 结果 |
| --- | --- | --- |
| 内核 | `uname -r`（三台） | 6.8.0-138-generic（GA，已 hold） |
| 服务端版本 | `lctl version`（mds1/oss1） | 2.17.0 |
| 客户端版本 | `lctl version`（client1） | 2.17.0_24_gf517bc4 |
| LNet | `sudo lctl list_nids` | 192.168.12.220/221/222@tcp |
| LNet 连通 | `sudo lnetctl ping` | 三机两两互通 |
| 目标注册 | `sudo lctl get_param mgs.MGS.live.*` | MDT0000 + OST0000 + OST0001 全部注册 |
| 挂载 | `mount \| grep lustre`（client1） | 192.168.12.220@tcp:/lustre on /mnt/lustre |
| 容量 | `lfs df -h` | MDT 23.5G + OST×2 47.6G×2 = **95.2G** |
| 条带化 | `lfs getstripe -v` | stripe_count=2，横跨 OST0000/0001 |
| IO | `dd` 写 512M | **114 MB/s** |
| 重启自愈 | 三台依次重启 | 全部自动恢复，无需干预 |

---

## 5. 实际部署中的踩坑与解决记录

| # | 现象 | 根因 | 解决 |
| --- | --- | --- | --- |
| 1 | 三台内核是 7.0.0-29-generic（HWE） | 镜像默认 HWE 内核线，Lustre/ZFS 不支持 | 装 6.8 GA → GRUB 切默认 → 重启 → hold → **purge 7.0**（§3.2） |
| 2 | `zfs-dkms` 安装卡死，dpkg 锁被占 | 为 7.0 内核构建 ZFS 2.2.2 失败（仅支持内核≤6.6） | 先 purge 7.0 再装 ZFS；卡死时 `pkill -f "dkms build -m zfs"` + `dpkg --configure -a` |
| 3 | `./configure --with-zfs` 报 missing zfs development headers | 缺 ZFS 用户态开发头文件 | `apt install libzfslinux-dev` |
| 4 | `make debs` 报 Unmet build dependencies | 缺 module-assistant / debhelper / quilt | `apt install module-assistant debhelper quilt` |
| 5 | GitHub `git clone` 极慢/卡住 | 直连被限速 | 走代理 `http://192.168.12.187:7897`（§3.5） |
| 6 | 普通用户 `modprobe lustre` 报 `Operation not permitted` | 内核模块加载需 root | 全部改用 `sudo` |
| 7 | sudo 下脚本 `$HOME` 变 /root | sudo 环境变量 | 脚本内用显式路径 `/home/<user>/...` |
| 8 | 方案写 `@tcp0`，实际显示 `@tcp` | Lustre 2.17 LNet 默认网络名 | 互 ping / 挂载统一用 `@tcp` |
| 9 | `lnet-setup.service` 重启后 failed | LNet 模块随挂载已加载，unit 执行时机 | 可加 `ExecStartPre=/usr/sbin/modprobe lnet`（不修也不影响） |

---

## 6. 遗留事项与后续建议

1. **扩容**：在 oss1 加一块数据盘建 `lustre-ost2`（`zpool create` + `mkfs.lustre --ost --index=2 --mgsnode=192.168.12.220@tcp` + mount），其他节点无需改动，容量自动并入。
2. **防火墙**：实验网络隔离未启用 ufw；生产环境建议 `sudo ufw allow 988/tcp` 并启用。
3. **HA**：本实验为单实例（MGS/MDT/OST 各一份）；生产应规划 Pacemaker 故障切换与 ZFS 冗余（mirror/RAID-Z）。
4. **版本说明**：ZFS 2.2.2 为 Ubuntu 自带版本；官方支持矩阵中 2.17 对应 ZFS 2.3.x（测试版本），如遇兼容问题以官方为准。

---

## 7. 参考资料与配套文档

| 资料 | 位置 / 地址 | 用途 |
| --- | --- | --- |
| 本文（06） | `docs/06-lustre-ubuntu24.04-3节点-部署方案.md` | 实际部署操作记录（本文） |
| 05 指南 | `docs/05-lustre-ubuntu24.04-部署指南.md` | 方法论与原理（版本选型、客户端路径、服务端编译、排障） |
| 04 指南 | `docs/04-lustre-虚拟机部署指南.md` | RHEL 系对照（命令一致性） |
| Lustre 官方 Wiki | https://wiki.lustre.org/ | Quick Start / Building / 运维手册 |
| LU-18010 | https://jira.whamcloud.com/browse/LU-18010 | Ubuntu 服务端构建依据（gcc-13、免补丁内核思路） |
| Whamcloud Support Matrix | https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix | 版本 × 发行版 × 内核 × ZFS 权威对照 |
| Lustre_Manual_cn | `docs/Lustre_Manual_cn_0.0.3.pdf` | 官方手册中文版（mkfs.lustre / lctl / lfs 参数） |
