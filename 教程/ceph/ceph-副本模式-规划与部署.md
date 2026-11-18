### 一、虚拟机数量与硬件配置要求

#### 1. 虚拟机数量
Ceph 副本模式（默认3副本）的核心要求是**副本跨节点分布**，保证单节点故障时数据不丢失，因此：
- **最低推荐（学习/测试首选）：3台虚拟机**
  每台同时承载 **MON（监控节点）+ MGR（管理节点）+ OSD（存储节点）** 三种角色，既满足 MON 高可用（3个MON符合Paxos过半机制，单MON故障集群仍正常运行），又满足3副本数据冗余要求，是最节省资源的标准测试方案。
- **极简体验（无高可用）：2台虚拟机**
  仅支持2副本，无法容忍单节点故障，仅适合快速体验功能，不推荐用于学习标准架构。
- **生产级角色分离：≥6台虚拟机**
  3台独立MON节点 + 3台以上独立OSD节点 + 可选MDS/RGW节点，适合生产环境，学习测试无需此配置。

#### 2. 单台虚拟机硬件配置
| 资源类型 | 最低可运行配置 | 推荐流畅配置 | 说明                                                         |
| -------- | -------------- | ------------ | ------------------------------------------------------------ |
| CPU      | 2核            | 4核          | 数据压缩、加密场景需更多CPU资源                              |
| 内存     | 3GB            | 6GB+         | 每个OSD进程建议至少分配1GB内存，MON/MGR额外占用约1GB，podman容器运行时也需要额外内存 |
| 系统盘   | 30GB           | 50GB         | 安装操作系统与Ceph软件包及容器镜像（约2~3GB）                |
| 数据盘   | 1块×20GB       | 1块×50GB+    | **必须是独立的裸磁盘**（不能是系统盘的分区），每个OSD对应1块数据盘 |
| 网络     | 千兆网卡       | 千兆/万兆    | 所有节点需在同一网段，网络互通                               |

#### 3. 可用容量计算
- 核心公式：**可用逻辑容量 = 所有数据盘总容量 ÷ 副本数**
- 示例：3台虚拟机，每台配1块50GB数据盘，3副本模式下：
  总原始容量 = 3 × 50GB = 150GB
  实际可用容量 ≈ 150GB ÷ 3 ≈ 50GB
- 注意：实际可用会扣除约10%的元数据、系统预留空间，属于正常情况。

---

### 二、详细搭建步骤
以下采用官方推荐的 `cephadm` 工具部署 **Ceph 19.x（Squid 稳定版）**，操作系统使用 **Ubuntu 24.04 LTS**，全程命令行操作。

> **为什么选 Squid 而不是 Reef？** Ceph Reef（18.x）的 Ubuntu 官方包针对的是 22.04（jammy），在 24.04（noble）上安装会遇到 Boost 等系统库的依赖冲突。Squid（19.x）是首个完整适配 Ubuntu 24.04 的稳定版本，推荐直接使用。

#### 前期规划
提前规划3台节点的基础信息，后续步骤以此为例：

| 主机名 | IP地址          | 角色        | 数据盘设备名 |
| ------ | --------------- | ----------- | ------------ |
| ceph1  | 192.168.12.176 | MON/MGR/OSD | /dev/sdb     |
| ceph2  | 192.168.12.90 | MON/MGR/OSD | /dev/sdb     |
| ceph3  | 192.168.12.169 | MON/MGR/OSD | /dev/sdb     |

> 数据盘设备名可通过 `lsblk` 命令确认，必须是未分区、未挂载的空白裸盘。
> 如果你的虚拟机用的是虚拟磁盘（如 QEMU/KVM 的 virtio），设备名可能是 `/dev/vdb`，请根据实际情况替换。

---

#### 步骤1：系统初始化
> **执行位置：ceph1、ceph2、ceph3（所有3台节点，逐一执行）**

所有节点统一执行基础环境配置，保证集群一致性。**以下命令需以 root 身份或加 sudo 执行。**

**1.1 设置对应主机名**

各节点分别执行对应的一条：
```bash
# 在 ceph1 执行
hostnamectl set-hostname ceph1

# 在 ceph2 执行
hostnamectl set-hostname ceph2

# 在 ceph3 执行
hostnamectl set-hostname ceph3
```

**1.2 配置主机名解析**

> **执行位置：ceph1、ceph2、ceph3（所有节点）**

```bash
cat >> /etc/hosts << 'EOF'
192.168.12.176 ceph1
192.168.12.90 ceph2
192.168.12.169 ceph3
EOF
```
> 注意：IP 地址请替换为你实际规划的值。如果虚拟机用 DHCP，建议先配置为静态 IP 再执行此步骤。

**1.3 禁用交换分区**（Ceph强制要求，否则会严重影响性能和稳定性）

> **执行位置：ceph1、ceph2、ceph3（所有节点）**

```bash
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```

**1.4 安装容器运行时**（cephadm 以容器方式运行所有 Ceph 守护进程）

> **执行位置：ceph1、ceph2、ceph3（所有节点）**

```bash
# Ubuntu 24.04 推荐使用 podman
apt update && apt install -y podman lvm2
```
> 也可以使用 `docker.io`，但 podman 是 RHEL/CentOS 生态的首选，与 cephadm 兼容性更好。两个选一个即可，不要同时装。

**1.5 配置时间同步**（Ceph对时间误差要求≤50ms，必须同步）

> **执行位置：ceph1、ceph2、ceph3（所有节点）**

```bash
apt install -y chrony
systemctl enable --now chrony
```
> Ubuntu 24.04 上服务名是 `chrony`（不是 `chronyd`），切记。

验证时间同步状态（所有节点）：
```bash
chronyc tracking
# 关注 "System time" 行，偏移量应在毫秒级别
```

**1.6 关闭防火墙**（测试环境简化，生产环境需开放对应端口）

> **执行位置：ceph1、ceph2、ceph3（所有节点）**

```bash
ufw disable
```

**1.7 安装基础依赖工具**

> **执行位置：ceph1、ceph2、ceph3（所有节点）**

```bash
apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
```

---

#### 步骤2：配置SSH免密登录
> **执行位置：仅在 ceph1 上执行**

cephadm 通过 SSH 管理集群节点，需要配置 ceph1 到所有节点（含自身）的 root 免密登录。

```bash
# 生成SSH密钥（如果还没有），一路回车即可
ssh-keygen -t rsa -N '' -f ~/.ssh/id_rsa

# 分发公钥到三台节点（根据提示输入各节点 root 密码）
ssh-copy-id root@ceph1
ssh-copy-id root@ceph2
ssh-copy-id root@ceph3
```

验证免密是否生效（在 ceph1 上执行，无需密码直接返回主机名即为成功）：
```bash
ssh root@ceph1 hostname
ssh root@ceph2 hostname
ssh root@ceph3 hostname
```

> **常见问题：`Permission denied (publickey,password)`**
>
> Ubuntu 24.04 默认禁止 root 通过 SSH 密码登录，即使密码正确也会被拒绝。如果在 `ssh-copy-id` 时遇到此报错，在 **ceph2 和 ceph3** 上执行以下修复：
> ```bash
> # 允许 root SSH 登录
> sed -i 's/^#PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
> sed -i 's/^PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
>
> # 确保密码认证开启
> sed -i 's/^#PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
> sed -i 's/^PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
>
> # 重启 SSH 服务
> systemctl restart sshd
> ```
> 修复后回到 ceph1 重新执行 `ssh-copy-id` 即可。免密配完后如想恢复安全设置，可把 `PermitRootLogin` 改回 `prohibit-password`。

---

#### 步骤3：安装 cephadm 并初始化集群
> **执行位置：仅在 ceph1 上执行**

**3.1 添加 Ceph Squid 官方软件源**
```bash
curl -fsSL https://download.ceph.com/keys/release.asc | \
  gpg --dearmor -o /usr/share/keyrings/ceph-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/ceph-archive-keyring.gpg] https://download.ceph.com/debian-squid/ $(lsb_release -sc) main" | \
  tee /etc/apt/sources.list.d/ceph.list

apt update
```
> `$(lsb_release -sc)` 在 Ubuntu 24.04 上会输出 `noble`，最终 repo 地址为 `https://download.ceph.com/debian-squid/ noble main`。

**3.2 安装 cephadm 和 ceph-common**
```bash
apt install -y cephadm ceph-common
cephadm version    # 确认版本号，应显示 19.x
```

**3.3 引导集群，启动第一个 MON + MGR 节点**
```bash
cephadm bootstrap --mon-ip 192.168.12.176
```
> 将 IP 替换为你 ceph1 节点的实际 IP。
>
> 首次执行会从 `quay.io` 拉取 Ceph 容器镜像（约 1.3GB），请确保网络通畅。如果拉取较慢，可提前配置 Docker/Podman 镜像加速。
>
> 执行完成后，终端会输出以下**重要信息，务必保存**：
> - **Dashboard URL**：`https://ceph1:8443`
> - **admin 用户名和初始密码**
>
> 如果忘记了密码，后续可通过以下命令重置：
> ```bash
> ceph dashboard ac-user-set-password admin -i <(echo "新密码")
> ```

**3.4 验证初始集群状态**
```bash
ceph -s
```
此时显示：1个 MON、1个 MGR（active），状态为 `HEALTH_WARN` — 属于正常现象（尚未添加其他节点和 OSD）。

**3.5 分发 cephadm 管理密钥到其他节点**

bootstrap 过程生成了 cephadm 自己的 SSH 密钥（`/etc/ceph/ceph.pub`），后续添加节点/部署服务都需要用它。虽然你已经在步骤2配了 root 密钥，但 cephadm 用的是**这套独立密钥**：

> **执行位置：ceph1**

```bash
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph2
ssh-copy-id -f -i /etc/ceph/ceph.pub root@ceph3
```
> 如果跳过这一步，后续 `ceph orch host add` 会报 `Permission denied`。

---

#### 步骤4：添加剩余节点到集群
> **执行位置：仅在 ceph1 上执行**

**4.1 将 ceph2、ceph3 加入集群主机列表**
```bash
ceph orch host add ceph2 192.168.12.90
ceph orch host add ceph3 192.168.12.169
```
> 显式指定 IP 更可靠，避免 DNS 解析问题。

**4.2 查看主机列表，确认三台都已接入**
```bash
ceph orch host ls
```

**4.3 扩展 MON 为 3 节点高可用**
```bash
ceph orch apply mon 3
```
cephadm 会自动为 ceph2、ceph3 各部署一个 MON 守护进程。等待 1~2 分钟。

**4.4 验证 MON 部署状态**
```bash
ceph mon stat
# 输出示例：e3: 3 mons at {ceph1=[v2:192.168.12.176:3300],ceph2=...,ceph3=...}
```

---

#### 步骤5：添加 OSD 数据盘

**5.1 确认并清空数据盘**

> **执行位置：ceph1、ceph2、ceph3（所有节点，逐一执行）**

确认数据盘为空白裸盘：
```bash
lsblk
# 找到对应的数据盘（如 /dev/sdb 或 /dev/vdb），确认没有分区和挂载
```

如果需要清空历史数据（**⚠️ 数据不可恢复，务必确认盘符正确**）：
```bash
# 以 /dev/sdb 为例，每台节点上执行
wipefs -a /dev/sdb
```

**5.2 查看集群可用的数据盘**

> **执行位置：ceph1**

```bash
ceph orch device ls
```

**5.3 批量添加 OSD**

> **执行位置：ceph1**

```bash
ceph orch apply osd --all-available-devices
```
该命令自动扫描所有已加入节点上的空白裸盘，每个盘创建一个 OSD。

> **常见问题：bootstrap 节点自身的盘未被识别**
>
> 如果 `ceph orch device ls` 中看不到 ceph1 自己的 `/dev/sdb`（如显示为 `Has a FileSystem`），说明该盘有残留分区或 cephadm 未将其标记为可用。先在 ceph1 上执行 `wipefs -a /dev/sdb` 清空，然后单独添加：
> ```bash
> ceph orch daemon add osd ceph1:/dev/sdb
> ```
> 推荐的稳妥做法是**三台都用精确命令逐一添加**，避免遗漏：
> ```bash
> ceph orch daemon add osd ceph1:/dev/sdb
> ceph orch daemon add osd ceph2:/dev/sdb
> ceph orch daemon add osd ceph3:/dev/sdb
> ```

等待 1~2 分钟，在 ceph1 上查看 OSD 状态：
```bash
ceph osd tree
```
输出中能看到 3 个 OSD 分别对应三台主机，状态为 `up` 即为添加成功。

---

#### 步骤6：验证集群最终状态

> **执行位置：ceph1**

```bash
ceph -s
```

健康状态为 **`HEALTH_OK`**，且包含以下关键信息即为搭建完成：

| 检查项 | 预期状态 |
| ------ | -------- |
| mon   | 3 daemons, quorum ceph1,ceph2,ceph3 |
| mgr   | 1 active + 1 standby |
| osd   | 3 osds: 3 up, 3 in |
| pools | 默认 `.mgr` 存储池，3副本模式 |

---

### 三、可选：功能测试（RBD块存储）
> **执行位置：以下命令均在 ceph1 上执行**

**1. 创建3副本存储池**
```bash
# 创建名为 rbd_pool 的存储池，PG数32（小规模测试适用）
ceph osd pool create rbd_pool 32 32

# 启用RBD块存储功能
ceph osd pool application enable rbd_pool rbd

# 验证存储池为3副本
ceph osd pool get rbd_pool size
# 应输出：size: 3
```

**2. 创建10GB的块设备镜像**
```bash
rbd create --size 10G rbd_pool/test_img
```

**3. 查看镜像信息**
```bash
rbd info rbd_pool/test_img
```

**4. 映射块设备到本地使用（可选）**
```bash
# 映射RBD镜像为本地块设备
rbd map rbd_pool/test_img

# 查看映射结果
rbd showmapped

# 格式化并挂载（首次使用）
mkfs.ext4 /dev/rbd0
mkdir -p /mnt/ceph-test
mount /dev/rbd0 /mnt/ceph-test

# 测试写入
echo "Ceph RBD test OK" > /mnt/ceph-test/test.txt
cat /mnt/ceph-test/test.txt

# 使用完毕卸载
umount /mnt/ceph-test
rbd unmap rbd_pool/test_img
```

---

### 四、Dashboard 管理界面

访问 bootstrap 过程中输出的 Dashboard 地址（默认 `https://ceph1:8443`），使用 admin 账号登录：

- **用户名**：`admin`
- **密码**：`kxf55g0ge4` 输出中显示的初始密码

> 首次登录会要求修改密码。注意 Dashboard 是通过自签名证书提供的 HTTPS 服务，浏览器会提示不安全，点击"高级"→"继续访问"即可。

如果无法访问，在 ceph1 上检查 Dashboard 服务状态：
```bash
ceph orch ps --daemon-type mgr
```

---

### 五、故障排查与重置

#### 常见问题

| 问题 | 原因 | 解决 |
|------|------|------|
| `ceph -s` 报 MON_CLOCK_SKEW | 节点时间不同步 | 在所有节点执行 `chronyc tracking`，确保已同步 |
| OSD 添加失败，状态 down | 数据盘有旧分区信息 | 在对应节点执行 `wipefs -a /dev/sdX`，然后在 ceph1 执行 `ceph orch daemon add osd hostname:/dev/sdX` |
| `apt install cephadm` 报依赖错误 | 使用了错误的 repo（Reef而非Squid） | 在 ceph1 确认 `/etc/apt/sources.list.d/ceph.list` 中是 `debian-squid` |
| 容器镜像拉取失败 | 网络不通或 quay.io 被墙 | 在 ceph1 配置 Podman 镜像加速，或手动 `podman pull quay.io/ceph/ceph:v19` |
| 节点添加失败 | SSH 免密未配置正确 | 在 ceph1 执行 `ssh root@ceph2 hostname` 验证免密，检查 `~/.ssh/known_hosts` |

#### 完全重置集群

若部署出错需要彻底重来：

```bash
# 1. 在 ceph1 上获取集群FSID
ceph fsid
# 记下输出的FSID，例如 abcdef12-3456-7890-abcd-ef1234567890

# 2. 在 ceph1 上清空集群
cephadm rm-cluster --fsid <FSID> --force

# 3. 在 ceph1、ceph2、ceph3 每台节点上清空 LVM 残留（逐一执行）
vgremove -f $(vgs --noheadings -o vg_name 2>/dev/null) 2>/dev/null
pvremove -f $(pvs --noheadings -o pv_name 2>/dev/null) 2>/dev/null

# 4. 重新从步骤1开始
```

---

### 六、注意事项

1. **数据盘必须是独立的裸磁盘**，不能使用系统盘的分区，否则 cephadm 不会识别。
2. **内存要给足**：Ubuntu 24.04 本身内存占用比 22.04 稍高，加上容器运行时和 Ceph 守护进程，建议每台 ≥4GB。
3. **容器运行时**：cephadm 依赖 podman（或 docker），Ubuntu 24.04 上推荐 podman。
4. **网络互通**：三台虚拟机必须在同一网段且互相可达，`ping` 通是前提。
5. **静态 IP**：建议所有节点配置静态 IP，避免 DHCP 变更导致集群分裂。
6. Ceph Squid 的完整官方文档：https://docs.ceph.com/en/squid/
