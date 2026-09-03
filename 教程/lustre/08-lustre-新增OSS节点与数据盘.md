# Lustre 集群扩容：新增 OSS 节点与数据盘操作指南（Ubuntu 24.04 + Lustre 2.17.0 + ZFS）

> 本文档基于 2026-08-20 实测的 3 节点集群（mds1/oss1/client1，192.168.12.220/221/222，Lustre 2.17.0 + ZFS 2.2.2，见 `docs/06`）编写，覆盖两种扩容场景：
> - **场景 A：全新增加一台 OSS 服务器**（完整流程：内核修正 → 服务端编译 → 建 ZFS 池 → 创建 OST → 自启验证）
> - **场景 B：在现有 oss1 上加一块数据盘**（快速路径：建池 → mkfs → 挂载，几分钟完成）
>
> 两种场景的本质相同——**都是创建新的 OST**。Lustre 的设计决定了扩容是"加目标"而非"改配置"：新 OST 挂载后由 MGS 自动感知、容量自动并入，其他节点（含客户端）**无需任何改动**。

---

## 目录

1. [结论先行（扩容原理与两种路径）](#1-结论先行)
2. [扩容前置检查](#2-扩容前置检查)
3. [场景 A：新增 OSS 节点](#3-场景-a新增-oss-节点oss2)
4. [场景 B：现有 oss1 加数据盘（快速路径）](#4-场景-b现有-oss1-加数据盘快速路径)
5. [扩容后：新数据如何利用新 OST](#5-扩容后新数据如何利用新-ost)
6. [常见问题与排障](#6-常见问题与排障)
7. [参考信息](#7-参考信息)

---

## 1. 结论先行

| 维度 | 说明 |
|------|------|
| 扩容原理 | 新增 OST（`mkfs.lustre --ost`）→ 挂载 → MGS 自动注册 → 容量与带宽自动并入，**全集群无感知** |
| 场景 A 耗时 | 全新节点约 **1~1.5 小时**（大头是服务端源码编译，约 40 分钟） |
| 场景 B 耗时 | 现有节点加盘约 **10 分钟** |
| 关键约束 | 新 OST 的 `--index` 必须为**当前集群最大 index + 1**（现有最大为 1，故从 2 开始），且**永不复用已删除的 index** |
| 扩容后注意 | 新文件可能落在新 OST（默认布局由 MDS 按空间自动选），**已有文件不会自动迁移**，需 `lfs migrate`（见 §5.2） |

**当前集群事实（来自 06 实测）**：

| 节点 | IP | 角色 | 现有目标 |
|------|-----|------|----------|
| mds1 | 192.168.12.220 | MGS + MDS | MGT、MDT0000 |
| oss1 | 192.168.12.221 | OSS | OST0000 / OST0001（index 0、1） |
| client1 | 192.168.12.222 | Client | 挂载 `192.168.12.220@tcp:/lustre` → /mnt/lustre |

- fsname：`lustre`；LNet 网络：`tcp`（NID 形如 `192.168.12.x@tcp`）；后端：ZFS，池命名 `lustre-ostN`
- 登录用户 sudo 密码：`1`；网卡：`ens33`；GitHub 需走代理 `192.168.12.187:7897`

---

## 2. 扩容前置检查

在任何操作前，先确认集群现状与新增硬件：

```bash
# [mds1] 确认当前已注册目标与最大 OST index
sudo lctl get_param mgs.MGS.live.*        # 期望列出 lustre-MDT0000 / OST0000 / OST0001

# [client1] 确认当前容量基线（扩容后对比用）
lfs df -h /mnt/lustre                     # 记录当前 MDT/OST 容量

# [新节点 或 oss1] 确认新数据盘为裸盘（未分区、未格式化、未挂载）
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
sudo fdisk -l /dev/sdX                    # 按实际设备名核对
```

- ✔ 验证：`lctl get_param mgs.MGS.live.*` 输出中最大 OST 编号为 1，则新 OST index 从 **2** 开始。
- ⚠️ 若之前曾删除过 OST，`lfs df -h` 会显示 index 空洞，新 index 取"当前活跃最大 index + 1"即可（删除的 index 不复用）。

---

## 3. 场景 A：新增 OSS 节点（oss2）

> 以下以新增 **oss2（IP 192.168.12.223，登录用户 user2，带 2 块数据盘 /dev/sdb、/dev/sdc）** 为例。IP、用户名、盘符请按实际替换。流程与 06 文档 §3.2~3.9 在 oss1 上的执行完全一致，差异仅在 index 起始值和主机名。

### 3.1 环境探查 `[oss2]`

```bash
ssh user2@192.168.12.223
cat /etc/os-release | head -3
uname -r                          # 若为 7.0.0-xx-generic（HWE 线），必须执行 §3.2
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE
sudo -n true && echo NOPASS || echo NEEDS_PASS
```

### 3.2 内核修正：切回 6.8 GA 并锁定 `[oss2]`（仅当内核为 7.0 HWE 时执行）

> 为什么必须做：Lustre 2.17 官方支持 Ubuntu 24.04 的 6.8 GA 内核线；ZFS 2.2.2 仅兼容内核 ≤6.6，在 7.0 内核上 `zfs-dkms` 编译必然失败。与 06 §3.2 完全相同。

```bash
# 1) 安装 GA 内核与头文件
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic
ls /boot/vmlinuz-*                # 确认出现 vmlinuz-6.8.0-xx-generic

# 2) 设置 GRUB 默认启动 6.8（菜单项以实际 update-grub 输出为准）
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT="Advanced options for Ubuntu>Ubuntu, with Linux 6.8.0-138-generic"/' /etc/default/grub
sudo update-grub
sudo reboot
uname -r                          # ✔ 期望 6.8.0-138-generic

# 3) 锁定内核 + 移除 HWE 7.0（顺序不可反，否则 zfs-dkms 卡死，见 06 §3.2 踩坑）
sudo apt-mark hold linux-image-generic linux-headers-generic
sudo apt purge -y linux-image-7.0.0-29-generic linux-headers-7.0.0-29-generic \
  linux-image-generic-hwe-24.04 linux-headers-generic-hwe-24.04 \
  linux-modules-7.0.0-29-generic
```

> 🔧 若已误装 `zfs-dkms` 导致 dpkg 卡死：`sudo pkill -f "dkms build -m zfs"` + `sudo dpkg --configure -a`。

### 3.3 基础环境 `[oss2]`

```bash
sudo hostnamectl set-hostname oss2

# /etc/hosts：三机映射 + 新节点自身（删除默认 127.0.1.1 行）
sudo tee -a /etc/hosts <<'EOF'
192.168.12.220  mds1
192.168.12.221  oss1
192.168.12.222  client1
192.168.12.223  oss2
EOF
sudo sed -i '/^127.0.1.1 /d' /etc/hosts
getent hosts mds1 oss1 client1 oss2

# 时间同步（Lustre 对时钟敏感）
sudo apt install -y chrony
sudo systemctl enable --now chrony

# 编译工具 + 与运行内核严格一致的头文件
sudo apt install -y git curl wget ca-certificates gnupg lsb-release software-properties-common \
  build-essential gcc make linux-headers-$(uname -r)
```

### 3.4 ZFS 与编译依赖 `[oss2]`

```bash
# 编译依赖（实测必需：libzfslinux-dev / module-assistant / debhelper / quilt，缺一失败）
sudo apt install -y build-essential gcc make flex bison pkg-config \
  zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
  libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
  python3-dev dpkg-dev \
  libzfslinux-dev module-assistant debhelper quilt

# ZFS（zfs-dkms 按当前 6.8 内核编译模块）
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs && echo ZFS_LOADED
zpool version                     # ✔ 期望 zfs-2.2.2-0ubuntu9.4
ls /usr/src | grep zfs            # ✔ 期望 /usr/src/zfs-2.2.2
```

### 3.5 服务端源码编译 `[oss2]`（约 40 分钟，后台执行）

```bash
# GitHub 直连慢，必须走代理
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
# 轮询：tail /tmp/lustre_make.log；产出 7 个 .deb（含 lustre-server-modules-6.8.0-xx-generic）
ls ~/lustre-release/debs/*.deb
```

### 3.6 安装服务端并加载模块 `[oss2]`

```bash
cd ~/lustre-release/debs
sudo dpkg -i *.deb
sudo depmod -a
sudo modprobe libcfs && echo LIBCFS_OK
sudo modprobe lustre && echo LUSTRE_OK
lctl version                      # ✔ 期望 2.17.0
```

### 3.7 LNet 配置与连通验证 `[oss2]`

```bash
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if ens33
sudo lctl list_nids               # ✔ 期望 192.168.12.223@tcp
sudo lnetctl ping 192.168.12.220@tcp    # ✔ 与 MGS(mds1) 互通，扩容前提
sudo lnetctl ping 192.168.12.221@tcp    # ✔ 与现有 OSS 互通
```

### 3.8 建 ZFS 池 `[oss2]`

```bash
# 2 块数据盘 → 2 个独立池（单盘单池，与 oss1 保持一致）
sudo zpool create lustre-ost2 /dev/sdb
sudo zpool create lustre-ost3 /dev/sdc
sudo zpool list                   # ✔ 出现 lustre-ost2 / lustre-ost3
```

> 提示：生产环境建议用 mirror/RAID-Z 冗余而非单盘池（当前实验环境与 06 一致为单盘池）。多盘合并建池也可，但单盘单池更利于按 OST 粒度管理。

### 3.9 创建并挂载 OST（index 从 2 开始）`[oss2]`

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
mount | grep lustre               # ✔ 两行 OST 挂载

# ✔ 注册验证 [mds1]：此时 MGS 已自动感知新 OST
sudo lctl get_param mgs.MGS.live.*   # 期望新增 lustre-OST0002 / lustre-OST0003
```

> 生产 HA 环境：mkfs 时建议加 `--servicenode=<主NID> --servicenode=<备NID>` 声明故障切换伙伴（实验环境与 06 一致未加）。

### 3.10 开机自启 `[oss2]`

```bash
# 1) fstab（服务端 OST 用 fstab + _netdev 即可，与 06 §3.13 一致）
# 追加：
#   lustre-ost2/ost /mnt/ost2 lustre defaults,_netdev 0 0
#   lustre-ost3/ost /mnt/ost3 lustre defaults,_netdev 0 0

# 2) ZFS 池写入 cachefile（开机自动导入）
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-ost2 lustre-ost3

# 3) LNet 开机自启（含 06 遗留建议的 ExecStartPre=modprobe lnet，避免 failed）
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
```

### 3.11 ✔ 扩容总验证

```bash
# [client1] 容量：应新增 2 个 OST
lfs df -h /mnt/lustre              # ✔ 期望出现 OST0002 / OST0003（如各 47.6G）

# [client1] 新文件显式使用新 OST
sudo mkdir -p /mnt/lustre/test2
sudo lfs setstripe -c 2 -S 4M /mnt/lustre/test2
sudo dd if=/dev/zero of=/mnt/lustre/test2/newfile bs=1M count=512 oflag=direct conv=fdatasync
lfs getstripe -v /mnt/lustre/test2/newfile   # ✔ obdidx 应含 2 和 3

# 重启自愈抽验 [oss2]
sudo reboot
# 重启后：zpool 自动导入、OST 自动挂载
sudo lctl dl | grep -cE ' UP '     # ✔ 应含 lustre-OST0002/0003 UP
mount | grep lustre
```

---

## 4. 场景 B：现有 oss1 加数据盘（快速路径）

> 不需要重新编译任何东西。核心三步：建池 → mkfs → 挂载。以下以新增一块盘 `/dev/sdd`（新 OST index=2）为例。

### 4.1 确认新盘 `[oss1]`

```bash
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,FSTYPE   # 确认 /dev/sdd 为裸盘、未被挂载
```

### 4.2 建池、mkfs、挂载 `[oss1]`

```bash
sudo zpool create lustre-ost2 /dev/sdd
sudo mkfs.lustre --fsname=lustre --ost --index=2 \
    --mgsnode=192.168.12.220@tcp --backfstype=zfs lustre-ost2/ost
sudo mkdir -p /mnt/ost2
sudo mount -t lustre lustre-ost2/ost /mnt/ost2
mount | grep lustre               # ✔ 出现 lustre-ost2/ost
```

### 4.3 自启与验证 `[oss1]` / `[client1]`

```bash
# fstab 追加：lustre-ost2/ost /mnt/ost2 lustre defaults,_netdev 0 0
sudo zpool set cachefile=/etc/zfs/zpool.cache lustre-ost2

# [client1] 验证
lfs df -h /mnt/lustre             # ✔ 期望新增 OST0002
```

> ⚠️ 若之前场景 A 已把 index 用到了 3，此处应取 `lfs df -h` 当前最大 index + 1（如 4），保证 index 全局唯一。

---

## 5. 扩容后：新数据如何利用新 OST

新增 OST 后**容量立即并入**，但"文件落在哪个 OST"由布局（layout）决定，三种情况：

### 5.1 新文件（默认行为）

- 目录/文件未显式设条带时，默认 `stripe_count=1`、`stripe_offset=-1`，由 MDS 自动选择 OST（倾向剩余空间充足的）。
- 新 OST 空间最大，新文件**大概率**自动落在新 OST，但不保证——需要确定性分布时显式指定：

```bash
# [client1] 目录级默认布局：该目录下新文件全部用满所有可用 OST（含新增的）
sudo lfs setstripe -c -1 -S 4M /mnt/lustre/dataset
# 或固定用前 N 个 OST（按需）
sudo lfs setstripe -c 4 -S 4M /mnt/lustre/dataset
```

### 5.2 已有文件迁移（`lfs migrate`）

**已有文件不会自动迁移到新 OST**。要让老数据利用新 OST 的容量/带宽，需显式迁移（会重写数据，IO 期间性能下降，建议低峰执行）：

```bash
# [client1] 迁移单个文件：重新条带到 2 个 OST（MDS 自动选，通常含新 OST）
lfs migrate -c 2 /mnt/lustre/test/bigfile

# 迁移整个目录（注意：目录内大量小文件时锁争用明显）
lfs migrate -c -1 /mnt/lustre/test

# 迁移后验证
lfs getstripe -v /mnt/lustre/test/bigfile   # ✔ obdidx 应含新 OST index
```

> 迁移前提：目标 OST 数量 ≤ 当前可用 OST 总数；`lfs migrate` 在线执行，文件可继续被读取（写期间短暂不可写）。

---

## 6. 常见问题与排障

| # | 现象 | 根因 | 解决 |
|---|------|------|------|
| 1 | 新节点 `./configure --with-zfs` 报 missing zfs development headers | 缺 ZFS 用户态开发头文件 | `sudo apt install libzfslinux-dev` |
| 2 | `make debs` 报 Unmet build dependencies | 缺 module-assistant / debhelper / quilt | `sudo apt install module-assistant debhelper quilt` |
| 3 | `zfs-dkms` 安装卡死、dpkg 锁被占 | 在 7.0 HWE 内核上构建 ZFS 失败 | 按 §3.2 先切 6.8 GA 并 purge 7.0 再装；卡死时 `pkill -f "dkms build -m zfs"` + `dpkg --configure -a` |
| 4 | GitHub clone 极慢/卡住 | 直连被限速 | 走代理 `http://192.168.12.187:7897` |
| 5 | 新 OST 挂载后 `lfs df -h` 不显示 | 挂载未生效 / index 冲突 | `mount \| grep lustre` 确认挂载；`sudo lctl get_param mgs.MGS.live.*` 确认注册；核对 index 唯一 |
| 6 | mkfs 报 index 已存在 | 复用了已删除的 index | 换用当前最大 index + 1 |
| 7 | 新文件没落到新 OST | 默认布局由 MDS 自动选，非确定 | 显式 `lfs setstripe -c -1` 或 `lfs migrate` |
| 8 | 重启后 LNet `failed` | lnet-setup.service 时序 | 按 §3.10 加 `ExecStartPre=/usr/sbin/modprobe lnet` |
| 9 | 客户端被 evict | 时钟漂移 / recovery 超时 | chrony 校时；`sudo lnetctl ping` 排查网络 |

---

## 7. 参考信息

| 资料 | 位置 / 地址 | 用途 |
|------|-------------|------|
| 本文（08） | `docs/08-lustre-新增OSS节点与数据盘.md` | 扩容操作指南（本文） |
| 06 部署记录 | `docs/06-lustre-ubuntu24.04-3节点-部署方案.md` | 原 3 节点部署全流程（§3.2~3.13 与本文场景 A 对应） |
| 05 部署指南 | `docs/05-lustre-ubuntu24.04-部署指南.md` | 方法论与原理（§4.3 后端选择、§5.4 ZFS 目标创建） |
| 01 知识点详解 | `docs/01-lustre-知识点详解.md` | 条带化参数、LDLM、扩容运维（§6.2 增删 OST、`lfs migrate`） |
| Lustre 官方 Wiki | https://wiki.lustre.org/ | mkfs.lustre / lfs migrate / OST 管理 |
| Whamcloud Support Matrix | https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix | 版本 × 发行版 × 内核 × ZFS 权威对照 |

---

*文档版本：扩容操作指南，基于 06 实测环境（Lustre 2.17.0 + ZFS 2.2.2 + 6.8.0-138-generic）。命令中 IP/盘符/用户名为示例，执行前请按实际环境核对。*
*最后更新：2026-08-21 | 作者：WorkBuddy 文档团队*
