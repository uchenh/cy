# Ceph 客户端远程挂载操作文档

> **任务目标**：在独立 Linux 客户端上安装 Ceph 工具，远程连接集群并使用 RBD 块存储。

---

## 一、环境判断

| 资源 | 最低要求 |
|------|---------|
| CPU / 内存 / 系统盘 | 1 核 / 2 GB / 20 GB |
| 网络 | 桥接模式，与集群同网段 |

**规划**：主机名 `client`，IP `192.168.12.237`（建议值，须避开已用地址），网关 `192.168.12.254`，DNS `202.96.209.5 / 202.96.199.133`。

---

## 二、前提条件

- [ ] 集群正常：`ceph -s` 显示 `HEALTH_OK`
- [ ] 客户端已装 Ubuntu 24.04，与集群网络互通（`ping 192.168.12.176` 通）

---

## 三、集群参考信息

| 项目 | 值 |
|------|-----|
| MON 地址 | 192.168.12.176 / .90 / .169（端口 6789） |
| FSID | `ceph fsid` 获取 |
| admin 密钥 | `cat /etc/ceph/ceph.client.admin.keyring` 获取 |

---

## 四、操作步骤

### 步骤 1：客户端基础配置（客户端执行）

```bash
hostnamectl set-hostname client

cat >> /etc/hosts << 'HOSTS'
192.168.12.176 ceph1
192.168.12.90  ceph2
192.168.12.169 ceph3
192.168.12.237 client
HOSTS

# 静态 IP（根据实际规划修改）
cat > /etc/netplan/01-netcfg.yaml << 'NETPLAN'
network:
  version: 2
  renderer: NetworkManager
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.12.237/24]
      routes:
        - to: default
          via: 192.168.12.254
      nameservers:
        addresses: [202.96.209.5, 202.96.199.133]
NETPLAN
netplan apply
```

**✔ 验证**：`ping -c 2 192.168.12.176` 通即可。

---

### 步骤 2：安装 ceph-common（客户端执行）

> ⚠ 客户端是 Ubuntu 24.04，同样用 jammy 源（Squid 无 noble 包），但 `ceph-common` 一般能直接装上。

```bash
echo "deb [trusted=yes] https://download.ceph.com/debian-squid/ jammy main" > /etc/apt/sources.list.d/ceph.list
apt update && apt install -y ceph-common
```

**✔ 验证**：`ceph --version` 输出 `ceph version 19.2.5 ... squid`。

> 报依赖冲突时备选：`apt install -y ceph-common --no-install-recommends`；或参考 `ceph部署-副本模式.md` 用容器包装脚本方案。

---

### 步骤 3：获取集群认证（ceph1 执行）

```bash
cat /etc/ceph/ceph.client.admin.keyring
# 记录输出中的 key 值，格式如：key = AQCDEf1234...==
```

---

### 步骤 4：配置客户端认证（客户端执行）

```bash
mkdir -p /etc/ceph

cat > /etc/ceph/ceph.conf << 'CEPHCONF'
[global]
fsid = <替换为 ceph fsid 输出的值>
mon host = 192.168.12.176, 192.168.12.90, 192.168.12.169
mon initial members = ceph1, ceph2, ceph3
CEPHCONF

cat > /etc/ceph/ceph.client.admin.keyring << 'KEYRING'
[client.admin]
    key = <替换为步骤 3 记录的实际 key 值>
KEYRING

chmod 644 /etc/ceph/ceph.conf
chmod 600 /etc/ceph/ceph.client.admin.keyring
```

---

### 步骤 5：验证连接（客户端执行）

```bash
ceph -s
```

**✔ 预期**：显示 `HEALTH_OK`、3 MON、3 OSD。

> 失败时检查：ceph.conf 的 MON IP、网络（`telnet 192.168.12.176 6789`）、客户端防火墙（`ufw disable`）。

---

### 步骤 6：远程使用 RBD（核心）

**6.1 集群端准备测试镜像**（ceph1 执行）：

```bash
ceph osd pool create client_pool 32 32
ceph osd pool application enable client_pool rbd
rbd create --size 5G client_pool/test_disk
```

**6.2 客户端操作**（客户端执行）：

```bash
# 查看远端镜像
rbd ls client_pool          # 应输出 test_disk

# 映射（需内核模块）
modprobe rbd
rbd map client_pool/test_disk   # 输出 /dev/rbd0

# 格式化 + 挂载（格式化仅首次）
mkfs.ext4 /dev/rbd0
mkdir -p /mnt/ceph-rbd && mount /dev/rbd0 /mnt/ceph-rbd

# 读写测试
echo "Ceph RBD remote mount test OK" > /mnt/ceph-rbd/test.txt
cat /mnt/ceph-rbd/test.txt

# 卸载
umount /mnt/ceph-rbd
rbd unmap client_pool/test_disk
```

**✔ 验证**：`rbd showmapped` 显示 `/dev/rbd0`；`df -h /mnt/ceph-rbd` 显示约 5G 容量。

---

### 步骤 7：开机自动挂载（可选）

```bash
cat > /usr/local/bin/mount-ceph-rbd.sh << 'SCRIPT'
#!/bin/bash
modprobe rbd 2>/dev/null
rbd map client_pool/test_disk 2>/dev/null
sleep 2
mount /dev/rbd0 /mnt/ceph-rbd 2>/dev/null
SCRIPT
chmod +x /usr/local/bin/mount-ceph-rbd.sh

cat > /etc/systemd/system/ceph-rbd-mount.service << 'SERVICE'
[Unit]
Description=Mount Ceph RBD device
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mount-ceph-rbd.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
SERVICE

systemctl daemon-reload && systemctl enable ceph-rbd-mount.service
```

---

## 五、常见问题

| # | 问题 | 解决 |
|---|------|------|
| 1 | `ceph -s` 连接超时 | 检查网络/防火墙，`telnet 192.168.12.176 6789` 测试 |
| 2 | `rbd map` 报 `sysfs write failed` | `modprobe rbd` |
| 3 | `rbd map` 报 `Module rbd not found` | 安装 `linux-modules-extra-$(uname -r)` |
| 4 | `mount: /dev/rbd0 is not a block device` | 先 `rbd showmapped` 确认映射成功 |
| 5 | `auth: incorrect key` | 检查 keyring 文件内容 |

---

## 六、参考

- Ceph 官方文档：https://docs.ceph.com/en/squid/rbd/
- 集群部署文档：`ceph部署-副本模式.md`
