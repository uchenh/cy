# Ceph 新增存储节点操作文档（通用模板）

> **任务目标**：向现有集群添加新存储节点，扩增容量。
> **文档版本**：v2.2 | 2026-08-03
> **使用方式**：复制本文档，把 `<NEW_HOSTNAME>` 替换为新节点主机名、`<NEW_IP>` 替换为新节点 IP。
> **关键约定**：① SSH 用**密钥分发**（Ubuntu 24.04 默认禁止 root 密码 SSH，`sshpass` 会失败）；② ceph orch 依赖集群密钥 `/etc/ceph/ceph.pub`，必须分发到新节点。
> **适用场景**：新节点仅完成 3 项准备（系统安装 + 静态 IP + 数据盘），其余配置（podman/chrony、SSH 密钥）按本模板步骤执行。

---

## 一、环境判断

需在 VMware/Proxmox 新建 **Ubuntu 24.04 LTS** 虚拟机，桥接模式与集群同网段。

| 资源 | 推荐配置 |
|------|---------|
| CPU / 内存 / 系统盘 | 4 核 / 6 GB / 50 GB |
| **数据盘** | **1 块 × 50GB+ 独立裸盘**（不分区不格式化，作新 OSD） |

**节点规划**：

| 项目 | 值 |
|------|-----|
| 主机名 | **`<NEW_HOSTNAME>`**（示例 ceph6） |
| IP | **`<NEW_IP>`**（示例 192.168.12.178，避开已用地址） |
| 网关 / DNS | 192.168.12.254 / 202.96.209.5·202.96.199.133 |
| 数据盘 | /dev/sdb |

> 使用前先 `ping <NEW_IP>` 确认无响应，并避开现有节点 IP。

---

## 二、前提条件（新节点当前已具备）

- [x] 集群正常：`ceph -s` 显示 `HEALTH_OK`
- [x] 新节点已装 Ubuntu 24.04 + 静态 IP，与集群互通
- [x] 已添加空白数据盘 `/dev/sdb`

> 剩余前置项（podman/chrony 安装、SSH 密钥分发）**无需提前准备**，会在下方操作步骤中完成。

---

## 三、操作步骤

### 步骤 1：系统初始化（`<NEW_HOSTNAME>` 执行）

```bash
hostnamectl set-hostname <NEW_HOSTNAME>

cat >> /etc/hosts << 'HOSTS'
192.168.12.176 ceph1
192.168.12.90  ceph2
192.168.12.169 ceph3
192.168.12.157 ceph5
<NEW_IP>       <NEW_HOSTNAME>
HOSTS

swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab

apt update && apt install -y podman lvm2 chrony
systemctl enable --now chrony
ufw disable
```

**✔ 验证**：`ping -c 1 ceph1` 通；`podman --version` 输出 4.x；`chronyc tracking` 偏移毫秒级。

---

### 步骤 2：配置 SSH 免密（密钥分发）

> ⚠ Ubuntu 24.04 允许 root 密钥登录、禁止密码登录，**不要用 sshpass 密码方式**。ceph.pub 必须分发（ceph orch 用它连接新节点）。

**2.1 在 ceph1 获取密钥**：
```bash
cat /etc/ceph/ceph.pub          # 集群管理密钥（必须）
cat /root/.ssh/id_rsa.pub       # ceph1 root 公钥（方便免密运维）
```

**2.2 写入新节点 authorized_keys**（方法 A/B 二选一）：

方法 A（ceph1 直推，前提 ceph1 已能连新节点）：
```bash
ssh-copy-id -i /etc/ceph/ceph.pub root@<NEW_IP>
ssh-copy-id root@<NEW_IP>
```

方法 B（新节点手动粘贴，最通用）：
```bash
sudo bash -c 'cat >> /root/.ssh/authorized_keys'   # 粘贴 2.1 两份公钥，Ctrl+D
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
```

**✔ 验证**（ceph1）：`ssh root@<NEW_HOSTNAME> hostname` 免密返回主机名。

---

### 步骤 3：加入集群（ceph1 执行）

```bash
ceph orch host add <NEW_HOSTNAME> <NEW_IP>
```

**✔ 验证**：`ceph orch host ls` 主机数量 +1；等 30~60 秒，`ceph orch ps --host <NEW_HOSTNAME>` 有 exporter 等 daemon。

---

### 步骤 4：添加 OSD（ceph1 执行）

```bash
# 新节点上先清空数据盘
wipefs -a /dev/sdb    # 在 <NEW_HOSTNAME> 执行

# ceph1 上查看设备 → 添加 OSD
ceph orch device ls | grep <NEW_HOSTNAME>   # AVAIL = Yes 表示可用
ceph orch daemon add osd <NEW_HOSTNAME>:/dev/sdb
```

**✔ 验证**：`ceph -s | grep osd` → osd 数量 +1，up/in 状态相同；`ceph osd tree` 有新主机。

> 新 OSD 自动触发数据再平衡，后台进行，不影响现有业务。

---

### 步骤 5：监控数据再平衡（ceph1 执行）

添加 OSD 后 `ceph -s` 短暂显示 `HEALTH_WARN`（remapped/recovering）属正常现象，无需干预：

```bash
ceph -w          # 观察恢复进度，Ctrl+C 退出
ceph df          # 查看容量变化
```

再平衡通常 **5~15 分钟** 完成，所有 PG 回到 `active+clean` 即恢复 `HEALTH_OK`：

```bash
ceph pg stat
```

---

### 步骤 6：最终验证（ceph1 执行）

| 检查项 | 预期 |
|--------|------|
| `ceph -s` | HEALTH_OK（再平衡后），mon 数量不变 |
| OSD | 数量 +1，up/in 状态相同 |
| 容量 | 原始/可用容量相应增加 |

> MON 数量保持不变（3 个已满足高可用）。

---

## 四、常见问题

| # | 问题 | 解决 |
|---|------|------|
| 1 | `ceph orch host add` 报 `Failed to connect` | authorized_keys 缺 ceph.pub → 重做步骤 2，确认免密 |
| 2 | 设备 `AVAIL = No` | 磁盘有残留签名 → 新节点执行 `wipefs -a /dev/sdb` |
| 3 | 新 OSD 为 `down` | 等待 30~60 秒；或 wipefs 后重新 add osd |
| 4 | `HEALTH_WARN` + remapped/recovering | 再平衡中，正常，等自动完成 |
| 5 | 新节点 daemon 起不来 | 确认 podman 已装、chrony 误差 ≤50ms |

---

## 五、注意事项

1. **数据盘必须是独立裸盘**，不能是系统盘分区
2. **磁盘容量建议与现有 OSD 一致**，否则容量分布不均
3. **podman + chrony 必须预装**，否则 daemon/OSD 无法启动
4. **再平衡期间性能受影响**，生产建议维护窗口操作
5. 添加后**无需重启集群**，cephadm 自动管理
6. 移除节点：`ceph orch host rm <NEW_HOSTNAME>`（需先清空其 OSD）
7. **扩容后不可逆**（删 OSD 会触发再平衡），生产操作需谨慎
8. **IP 必须唯一**：选 `<NEW_IP>` 前先 `ping` 确认无冲突
9. 加多台节点时，重复步骤 1~4 即可，全部添加完统一再平衡
