# CephFS 文件存储挂载与使用指南

> **适用集群**：Ceph Squid 19.x + Ubuntu 24.04，4 节点（ceph1/2/3/5），4 OSD，240 GiB
> **前置文档**：`ceph部署-副本模式.md` — 集群部署流程

## 目录

- [一、CephFS 概述](#一cephfs-概述)
- [二、前提条件](#二前提条件)
- [三、部署 MDS 服务](#三部署-mds-服务)
- [四、挂载方式总览](#四挂载方式总览)
- [五、内核客户端挂载（mount -t ceph）](#五内核客户端挂载mount--t-ceph)
- [六、用户空间客户端挂载（ceph-fuse）](#六用户空间客户端挂载ceph-fuse)
- [七、自动挂载与排查](#七自动挂载与排查)
- [八、验证与操作](#八验证与操作)
- [九、高级功能](#九高级功能)
- [十、故障排查](#十故障排查)
- [十一、卸载方法](#十一卸载方法)

---

## 一、CephFS 概述

### 1.1 什么是 CephFS

CephFS 是 Ceph 提供的 **POSIX 兼容**分布式文件系统，底层复用同一 RADOS 存储池。多台客户端可同时挂载同一个 CephFS 实现**共享读写**，行为类似 NFS 但无单点瓶颈。

### 1.2 核心组件

```
客户端（ceph-fuse / 内核 mount -t ceph）
        │ 文件请求
        ▼
MDS 元数据服务 ──→ cephfs_metadata 池（目录树/权限）
        │
        ▼
cephfs_data 池（文件内容） ──→ RADOS（3 副本）
```

MDS 职责：维护目录树（inode/dentry）、与客户端协商权限（capability）、元数据缓存加速。MDS 默认单线程，可启用 multiple active MDS 提升并发。

### 1.3 三种存储对比

| 特性 | RBD（块） | CephFS（文件） | RGW（对象） |
|------|----------|--------------|-------------|
| 访问接口 | 块设备 /dev/rbdX | POSIX 文件系统 | S3 / Swift API |
| 共享方式 | 单机独占 | **多机共享读写** | HTTP RESTful |
| 适用场景 | VM 磁盘、数据库 | 共享目录、容器卷 | 图片、备份、静态文件 |
| 额外服务 | 无 | **需 MDS** | 需 RGW |
| 客户端依赖 | 内核 rbd 模块 | ceph-fuse / 内核 ceph 模块 | HTTP 库 |

### 1.4 存储池布局

| 存储池 | 用途 | 副本 | 建议 PG |
|--------|------|------|---------|
| `cephfs_metadata` | 文件名、目录结构、权限 | 3 | 32（元数据池不宜过大） |
| `cephfs_data` | 文件实际内容 | 3 | OSD 数 × 100 ÷ 副本数 |

---

## 二、前提条件

### 2.1 集群要求

- [ ] Ceph 集群运行正常：`ceph -s` 显示 `HEALTH_OK`
- [ ] 集群至少 2 个 OSD
- [ ] MON 节点网络可达（客户端能访问 6789 / 3300 端口）

### 2.2 客户端选择

| 客户端 | 依赖包 | 性能 | 本集群状态 |
|--------|--------|------|-----------|
| **ceph-fuse**（推荐） | `ceph-fuse` | 中（用户态 FUSE） | 🔧 ✅ 已验证 |
| 内核客户端（备选） | `ceph-common` | 高（内核态） | 🔧 ❌ kernel 7.0 下不可用 |

> 🔧 **本集群实测**：内核 7.0.0 + Ceph Squid 下 `mount -t ceph` 报 `libceph: socket error on write`，请用 **ceph-fuse**。
> ceph-fuse 需从 **Ubuntu 官方源**安装（不要走 ceph 的 jammy 源，依赖冲突）。

### 2.3 本集群环境速查

| 项目 | 值 |
|------|-----|
| 节点 / OSD 数 | 🔧 4 节点（ceph1/2/3/5）/ 4 OSD |
| MON 地址 | `192.168.12.176` / `.90` / `.169` |
| MON 端口 | 6789（v1）/ 3300（v2） |
| 内核版本 | 🔧 7.0.0-generic |
| 总容量 / 可用 | 🔧 240 GiB / ~199 GiB |
| 文件系统名 | `myfs` |
| 元数据池 / 数据池 | `cephfs_metadata`（PG 32）/ `cephfs_data`（PG 128） |
| 挂载点 | `/mnt/cephfs` |

---

## 三、部署 MDS 服务

> **执行位置**：ceph1。

```bash
# 1. 创建存储池（PG：数据池 128，元数据池 32）
ceph osd pool create cephfs_metadata 32 32
ceph osd pool create cephfs_data 128 128

# 2. 创建文件系统
ceph fs new myfs cephfs_metadata cephfs_data

# 3. 部署 MDS（3 个：1 active + 2 standby）
ceph orch apply mds myfs --placement="3 ceph1 ceph2 ceph3"
```

**✔ 验证**：

```bash
ceph fs ls
# name: myfs, metadata pool: cephfs_metadata, data pools: [cephfs_data]

ceph mds stat
# myfs:1 {0=ceph1.xxxxxx=up:active} 2 up:standby
# 解读：myfs:1 = 1 个 active MDS；2 up:standby = 2 个备用（故障自动接管）
```

---

## 四、挂载方式总览

```
本集群（Ubuntu 24.04 + Kernel 7.0 + Squid 19.x）
    └──→ 🔧 推荐：ceph-fuse（内核客户端存在兼容问题）
```

| 对比项 | 内核客户端 | ceph-fuse |
|--------|-----------|-----------|
| 安装依赖 | `ceph-common` | `ceph-fuse` |
| 性能 | ★★★★★ | ★★★☆☆ |
| 内核版本要求 | 需内核 ceph 模块 | 无 |
| 🔧 本集群状态 | ❌ kernel 7.0 不可用 | ✅ 已验证 |

---

## 五、内核客户端挂载（mount -t ceph）

> ⚠️ **本集群实测不兼容**（`socket error on write`），以下仅作其他兼容环境参考，本集群请直接用「六、ceph-fuse」。

```bash
# 客户端安装
apt install -y ceph-common

# 认证文件（/etc/ceph/ceph.conf + ceph.client.admin.keyring，从集群 ceph1 复制）
mkdir -p /mnt/cephfs
mount -t ceph 192.168.12.176:6789:/ /mnt/cephfs -o name=admin
# 多 MON 容错：192.168.12.176,192.168.12.90,192.168.12.169:6789:/
# 免 keyring 文件：-o name=admin,secret=<密钥>
```

---

## 六、用户空间客户端挂载（ceph-fuse）

> 🔧 **本集群推荐方案**，实际部署已验证通过。

### 6.1 安装

```bash
# 🔧 必须从 Ubuntu 官方源安装（先移除 ceph 源避免依赖冲突）
rm -f /etc/apt/sources.list.d/ceph.list
apt update && apt install -y ceph-fuse

# 装完如需恢复 ceph 源
echo "deb [trusted=yes] https://download.ceph.com/debian-squid/ jammy main" > /etc/apt/sources.list.d/ceph.list
```

### 6.2 准备认证文件

在客户端创建 `/etc/ceph/ceph.conf` 和 `ceph.client.admin.keyring`（从集群 ceph1 复制）：

```bash
mkdir -p /etc/ceph
# ceph.conf：
# [global]
# fsid = <ceph fsid>
# mon host = 192.168.12.176, 192.168.12.90, 192.168.12.169
# ceph.client.admin.keyring：从 ceph1 的 /etc/ceph/ 直接拷贝即可
chmod 644 /etc/ceph/ceph.conf
chmod 600 /etc/ceph/ceph.client.admin.keyring
```

### 6.3 挂载

```bash
mkdir -p /mnt/cephfs
ceph-fuse /mnt/cephfs                                    # 默认读 keyring 认证
ceph-fuse /mnt/cephfs -m 192.168.12.176:6789             # 指定 MON
ceph-fuse /mnt/cephfs -n client.admin --keyring /etc/ceph/ceph.client.admin.keyring
# 多 MON：-m 192.168.12.176:6789,192.168.12.90:6789,192.168.12.169:6789
```

**✔ 验证**：

```bash
mount | grep cephfs
# ceph-fuse on /mnt/cephfs type fuse.ceph-fuse (rw,nosuid,nodev,relatime)
df -h /mnt/cephfs
```

### 6.4 推荐参数组合

```bash
ceph-fuse /mnt/cephfs -m 192.168.12.176:6789 -n client.admin \
  --keyring /etc/ceph/ceph.client.admin.keyring --noatime
```

| 常用参数 | 说明 |
|----------|------|
| `-m` | 指定 MON 地址（默认自动发现） |
| `-n` | 客户端 ID（默认 client.admin） |
| `--keyring` | 指定 keyring 文件 |
| `--client_mountpoint` | 挂载子目录（如 `/data/project-a`） |
| `-r` / `--noatime` | 只读 / 不更新访问时间 |

---

## 七、自动挂载与排查

### 7.1 systemd 自动挂载（推荐）

```bash
cat > /etc/systemd/system/cephfs-mount.service << 'EOF'
[Unit]
Description=CephFS Mount via ceph-fuse
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStartPre=/usr/bin/mkdir -p /mnt/cephfs
ExecStart=/usr/bin/ceph-fuse /mnt/cephfs --name client.admin --foreground
ExecStop=/usr/bin/fusermount -u /mnt/cephfs
Restart=on-failure
RestartSec=10
TimeoutStartSec=180

[Install]
WantedBy=multi-user.target
EOF

# 开机自动加载 fuse 内核模块（关键！否则 ceph-fuse 报错退出）
echo fuse > /etc/modules-load.d/fuse.conf

systemctl daemon-reload
systemctl enable cephfs-mount.service
systemctl start cephfs-mount.service
```

> ⚠ **关键点**：必须先 `modprobe fuse` 加载 fuse 模块，否则 ceph-fuse 会报
> `Ignoring invalid max threads value 4294967295 > max (100000)` 并退出，挂载点消失。

### 7.2 挂载消失的排查

如果发现 File Browser 看不到文件（"It feels lonely here..."），按顺序排查：

```bash
mount | grep cephfs          # 1. 挂载是否还在
ps aux | grep ceph-fuse      # 2. 进程是否存活
systemctl status cephfs-mount.service  # 3. 服务状态
lsmod | grep fuse            # 4. fuse 模块是否加载
journalctl -u cephfs-mount -n 30       # 5. 日志
modprobe fuse && systemctl restart cephfs-mount.service   # 6. 修复
```

---

## 八、验证与操作

```bash
# 基本读写
mkdir -p /mnt/cephfs/data
echo "CephFS test file" > /mnt/cephfs/data/hello.txt
cat /mnt/cephfs/data/hello.txt
dd if=/dev/zero of=/mnt/cephfs/data/test.bin bs=1M count=100   # 大文件测试

# 多节点共享验证（ceph1 写入 → 客户端读取）
echo "shared data" > /mnt/cephfs/shared.txt

# 状态查看
mount | grep cephfs
ceph fs status myfs
ceph mds stat
ceph tell mds.0 client ls
```

---

## 九、高级功能

```bash
# 挂载子目录
ceph-fuse /mnt/project-a --client_mountpoint=/data/project-a

# 只读挂载
ceph-fuse /mnt/cephfs-ro -r

# CephFS 快照（需先启用）
ceph fs set myfs allow_new_snaps true
mkdir /mnt/cephfs/.snap/snap-20260727        # 创建快照
ls -la /mnt/cephfs/.snap/                    # 查看
cp -a /mnt/cephfs/.snap/snap-20260727/* /mnt/cephfs/   # 回滚

# 配额（需安装 attr 包）
setfattr -n ceph.quota.max_bytes -v 107374182400 /mnt/cephfs/data   # 100GiB
setfattr -n ceph.quota.max_files -v 100000 /mnt/cephfs/data
getfattr -n ceph.quota.max_bytes /mnt/cephfs/data

# 客户端细粒度授权（ceph1 执行）
ceph fs authorize myfs client.app1 / rw /data/app1 rw
# 输出的 key 给 app1 客户端挂载：-o name=app1,secret=<输出的 key>
```

---

## 十、故障排查

| # | 错误信息 | 原因 | 解决 |
|---|---------|------|------|
| 1 | `no mds server is up` | MDS 未部署/宕机 | `ceph mds stat` 确认有 active |
| 2 | `Connection refused` | MON 端口不通 | `telnet 192.168.12.176 6789` |
| 3 | `auth: incorrect key` | keyring 密钥不匹配 | `ceph auth get client.admin` 核对 |
| 4 | `-13 Permission denied` | 认证失败/无权限 | 检查 keyring 内容与权限 |
| 5 | `no active rank 0` | MDS 未就绪 | 等 30 秒重试 |
| 6 | `read-only file system` | 集群健康异常 | `ceph -s` 修复降级 OSD/PG |
| 7 | 🔧 `socket error on write`（dmesg） | Kernel 7.0 与 Squid 协议不兼容 | 改用 `ceph-fuse` |
| 8 | 挂载消失、fuse 报错退出 | fuse 模块未加载 | `modprobe fuse` + 配置 `/etc/modules-load.d/fuse.conf` |

**诊断思路**：网络通？(`telnet MON 6789`) → MDS 活？(`ceph mds stat`) → 认证对？(`ceph auth get client.admin`) → 内核报错？(`dmesg | grep ceph`，有 `socket error` 就换 ceph-fuse)。

**查看 MDS 日志**：

```bash
cephadm enter mds.ceph1
tail -100 /var/log/ceph/ceph-mds.ceph1.log
exit
```

**客户端会话管理**：

```bash
ceph tell mds.0 session ls
ceph tell mds.0 session evict id:<session_id>   # 强制断开
```

---

## 十一、卸载方法

```bash
# ceph-fuse / 内核客户端通用
umount /mnt/cephfs
# 失败时：lsof /mnt/cephfs 查占用 → umount -f /mnt/cephfs（强制）
# ceph-fuse 专用：fusermount -u /mnt/cephfs

# 清理 systemd 自动挂载
systemctl stop cephfs-mount.service
systemctl disable cephfs-mount.service
```

---

## 参考信息

| 项目 | 位置 |
|------|------|
| 集群部署文档 | `ceph部署-副本模式.md` |
| 客户端远程挂载（RBD） | `ceph-client-mount.md` |
| 新增节点文档 | `ceph-add-node.md` |
| CephFS 官方文档 | https://docs.ceph.com/en/squid/cephfs/ |
