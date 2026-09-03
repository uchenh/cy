# Ceph 19.x (Squid) 部署精简版

> **适用环境**：Ubuntu 24.04 LTS × 3 节点，Ceph Squid 19.x（cephadm 容器化部署）
> **总耗时**：约 30~60 分钟（取决于网络）

---

## 一、环境规划

| 主机名 | IP | 角色 | 数据盘 |
|--------|-----|------|--------|
| ceph1 | 192.168.12.176 | MON / MGR / OSD | /dev/sdb |
| ceph2 | 192.168.12.90  | MON / MGR / OSD | /dev/sdb |
| ceph3 | 192.168.12.169 | MON / MGR / OSD | /dev/sdb |

**账号**：`ps` / 密码 `1`（sudo 用户）；`root` 密码初始 `1`（本章节中开启 SSH 登录）。

**代理**（国内访问 quay.io 需要）：`http://192.168.12.187:7897`，替换为你实际地址。

---

## 二、部署前准备

1. **添加数据盘**：虚拟机平台给三台机器各加一块 ≥50GB 空白盘（不分区不格式化）。登录各节点确认识别：

```bash
lsblk   # 看到无分区的 sdb 即可
```

2. **网络互通**：三台互 ping 通（任一节点）：

```bash
ping -c 2 192.168.12.176 && ping -c 2 192.168.12.90 && ping -c 2 192.168.12.169
```

---

## 三、系统初始化（三台节点逐一执行）

> 以下命令在 ceph1、ceph2、ceph3 **每台都执行**（hostname 命令按对应节点改）。

```bash
# 3.1 主机名（每台节点改成自己的名字）
hostnamectl set-hostname ceph1 

# 3.2 hosts 解析
cat >> /etc/hosts << 'HOSTS'
192.168.12.176 ceph1
192.168.12.90 ceph2
192.168.12.169 ceph3
HOSTS

# 3.3 禁用交换分区
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab

# 3.4 容器运行时 + 基础工具
apt update && apt install -y podman lvm2 chrony curl

# 3.5 时间同步 + 关防火墙
systemctl enable --now chrony
ufw disable
```

**✔ 本章验证**：

```bash
podman --version        # 输出版本号
chronyc tracking        # System time 偏移毫秒级
free -m | grep Swap     # total 为 0
```

---

## 四、配置 root SSH 登录（三台节点逐一执行）

> cephadm 需要 root 权限，Ubuntu 24.04 默认禁止 root SSH，需先用 ps 用户提权开启。

```bash
# 用 ps 用户执行（sudo 提权）
echo 'root:1' | sudo chpasswd
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart ssh
```

**✔ 验证**（ceph1 上测三台）：

```bash
ssh root@192.168.12.176 hostname   # 输入密码 1，返回 ceph1
ssh root@192.168.12.90 hostname    # 返回 ceph2
ssh root@192.168.12.169 hostname   # 返回 ceph3
```

---

## 五、ceph1 SSH 免密（ceph1 执行）

```bash
# 生成密钥
ssh-keygen -t rsa -N '' -f /root/.ssh/id_rsa

# 安装 sshpass 并分发公钥到三台节点
apt install -y sshpass
sshpass -p '1' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.12.176
sshpass -p '1' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.12.90
sshpass -p '1' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.12.169
```

**✔ 验证**：`ssh root@ceph2 hostname` 免密直接返回 ceph2。

---

## 六、安装 cephadm + bootstrap（ceph1 执行）

> ⚠ 两个坑：① Squid 没有 noble 包，用 jammy 源；② 新版 cephadm 是 zipapp，必须用包装脚本调用。

```bash
# 6.1 添加 Ceph 源（trusted=yes 跳过 GPG，SSH 无 TTY 导不了密钥）
echo "deb [trusted=yes] https://download.ceph.com/debian-squid/ jammy main" > /etc/apt/sources.list.d/ceph.list

# 6.2 安装
apt update && apt install -y cephadm

# 6.3 创建包装脚本（必须！）
printf '#!/bin/bash\nexec python3 /usr/sbin/cephadm "$@"\n' > /usr/local/bin/cephadm
chmod +x /usr/local/bin/cephadm

# 6.4 代理（写进 bashrc 让后续命令都生效）
cat >> /root/.bashrc << 'PROXY'
export http_proxy="http://192.168.12.187:7897"
export https_proxy="http://192.168.12.187:7897"
PROXY
source /root/.bashrc

# 6.5 bootstrap（拉 ~1.3GB 镜像，5~15 分钟）
cephadm bootstrap --mon-ip 192.168.12.176
```

**✔ 验证 & 保存**：`cephadm version` 输出版本号；bootstrap 完成后**保存终端输出的 Dashboard 初始密码**（忘记可用 12.3 节命令重置）。

```bash
# 6.6 检查初始状态（此时 /usr/bin/ceph 还没建，必须用 cephadm shell）
cephadm shell ceph -s
# 应有 1 MON、1 MGR、HEALTH_WARN、0 OSD

# 6.7 分发集群密钥到 ceph2/ceph3
sshpass -p '1' ssh-copy-id -f -i /etc/ceph/ceph.pub -o StrictHostKeyChecking=no root@192.168.12.90
sshpass -p '1' ssh-copy-id -f -i /etc/ceph/ceph.pub -o StrictHostKeyChecking=no root@192.168.12.169
```

---

## 七、添加节点 + 扩展 MON（ceph1 执行）

```bash
# 7.1 添加节点
cephadm shell ceph orch host add ceph2 192.168.12.90
cephadm shell ceph orch host add ceph3 192.168.12.169

# 7.2 扩展 MON 到 3 个（自动部署到 ceph2/ceph3）
cephadm shell ceph orch apply mon 3
```

**✔ 验证**（等 1~2 分钟）：`cephadm shell ceph orch host ls` 显示 3 hosts；`cephadm shell ceph mon stat` 显示 3 mons。

---

## 八、添加 OSD（ceph1 执行）

```bash
# 8.1 清空三台节点的数据盘（每台节点执行一次）
wipefs -a /dev/sdb

# 8.2 逐个添加 OSD
cephadm shell ceph orch daemon add osd ceph1:/dev/sdb
cephadm shell ceph orch daemon add osd ceph2:/dev/sdb
cephadm shell ceph orch daemon add osd ceph3:/dev/sdb
```

**✔ 验证**：`cephadm shell ceph -s` → `osd: 3 osds: 3 up, 3 in`，health 变 `HEALTH_OK`（有新 OSD 会自动再平衡，等几分钟）。

> 若提示 `3 pool(s) do not have an application enabled`：`cephadm shell ceph osd pool application enable <池名> rbd`

---

## 九、部署后配置（ceph1 执行）

> `ceph-common`（ceph/rbd 命令）在 noble 上装不上（jammy 包依赖冲突），用包装脚本替代。

```bash
# 9.1 ceph 命令包装
cat > /usr/bin/ceph << 'CEPH'
#!/bin/bash
exec cephadm shell -- ceph "$@"
CEPH
chmod +x /usr/bin/ceph

# 9.2 rbd 命令包装（map 系列需挂载宿主机 /sys /dev）
cat > /usr/bin/rbd << 'RBD'
#!/bin/bash
NEED_MOUNT="map showmapped unmap"
FIRST=$1
if echo "$NEED_MOUNT" | grep -qw "$FIRST"; then
    exec cephadm shell --mount /sys:/sys --mount /dev:/dev -- rbd "$@"
else
    exec cephadm shell -- rbd "$@"
fi
RBD
chmod +x /usr/bin/rbd

# 9.3 加载 rbd 内核模块（仅需要 rbd map 时才需要）
modprobe rbd && echo 'rbd' > /etc/modules-load.d/rbd.conf
```

**✔ 验证**：`ceph -s` 正常输出集群状态（首次较慢，之后复用容器很快）。

---

## 十、RBD 功能测试（ceph1 执行）

```bash
# 创建池 + 镜像
ceph osd pool create mypool 32 32
ceph osd pool application enable mypool rbd
rbd create --size 5G mypool/my-disk

# 映射 + 格式化 + 挂载
rbd map mypool/my-disk
mkfs.ext4 /dev/rbd0
mkdir -p /mnt/ceph-rbd && mount /dev/rbd0 /mnt/ceph-rbd

# 读写测试
echo "Hello Ceph RBD!" > /mnt/ceph-rbd/test.txt && cat /mnt/ceph-rbd/test.txt

# 卸载
umount /mnt/ceph-rbd
rbd unmap mypool/my-disk
```

**✔ 验证**：`rbd ls mypool` 输出 `my-disk`；`df -h /mnt/ceph-rbd` 显示约 4.9G 挂载成功。

---

## 十一、Dashboard

- **地址**：`https://192.168.12.176:8443/`（自签名证书，浏览器点"高级→继续前往"）
- **账号**：`admin` / bootstrap 输出的随机密码
- **密码重置**（忘记或太弱时）：

```bash
echo "CephAdmin@2026" | cephadm shell -- ceph dashboard ac-user-set-password admin -i -
```

> Ceph 19 强密码策略：至少 8 位，需含大写字母+数字，`123456` 会被拒。

---

## 十二、常见问题速查

| # | 错误 | 解决 |
|---|------|------|
| 1 | `Permission denied (publickey,password)` | root SSH 未开启，回第四章用 ps 用户改配置 |
| 2 | `apt update` 404 | apt 源写成了 noble，应写 `jammy main` |
| 3 | `cephadm: command not found` | 未创建 6.3 节包装脚本 |
| 4 | `gpg: cannot open '/dev/tty'` | 用 `[trusted=yes]` 跳过 GPG |
| 5 | `rbd: sysfs write failed` | `modprobe rbd` |
| 6 | bootstrap 拉镜像极慢 | 配置代理后重跑 |
| 7 | Dashboard 密码太弱 | 用 8 位以上含大小写+数字的密码 |

---

## 十三、完全重置

```bash
# ceph1：清空集群（记下 FSID）
ceph fsid
cephadm rm-cluster --fsid <FSID> --force

# 三台节点：清理 LVM 残留 + 数据盘
vgremove -f $(vgs --noheadings -o vg_name 2>/dev/null) 2>/dev/null
pvremove -f $(pvs --noheadings -o pv_name 2>/dev/null) 2>/dev/null
wipefs -a /dev/sdb
rm -f /etc/apt/sources.list.d/ceph.list
```

然后从**第三章**重新开始。

