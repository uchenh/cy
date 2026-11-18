# Ceph部署与使用手册


<!-- 第 1 页 -->

Ceph 分布式存储部署与使用手册
Ceph Squid 19.x（cephadm 容器化部署）· Ubuntu 24.04 LTS · 副本模式（3 副本）


<!-- 第 2 页 -->

1
目 录
第一部分 部署总览 .............................................................. 2
第二部分 集群部署（副本模式） ................................................... 4
第三部分 集群扩容：新增存储节点 ................................................. 9
第四部分 客户端接入：RBD 远程挂载 .............................................. 12
第五部分 CephFS 文件存储 ...................................................... 16
第六部分 RGW 对象存储（S3 兼容 API） ........................................... 22
第七部分 Web 管理界面 ......................................................... 28
附录 ......................................................................... 33


<!-- 第 3 页 -->

2
第一部分 部署总览
1.1 集群环境规划
集群初始为 3 节点部署，后按第三部分流程扩容加入 ceph5（4 节点 / 4 OSD）。
项目 值
初始节点 ceph1 = 192.168.12.176、ceph2 = 192.168.12.90、ceph3 = 192.168.12.169（均为
MON / MGR / OSD）
扩容节点 ceph5 = 192.168.12.157（OSD）
独立客户端 client = 192.168.12.237（第四部分使用）
数据盘 各节点 /dev/sdb（≥50GB 独立裸盘，不分区不格式化）
网关 / DNS 192.168.12.254 / 202.96.209.5、202.96.199.133
操作系统 / 内核 Ubuntu 24.04 LTS / 7.0.0-generic ?
Ceph 版本 Squid 19.x，cephadm 容器化部署
集群容量 4 OSD 时期总计 240 GiB，可用约 199 GiB ?
FSID 用 ceph fsid 查看（示例集群：a2173549-84bf-11f1-8f8e-000c29c6cc12）
MON 端口 6789（v1 协议）/ 3300（v2 协议）
认证文件 /etc/ceph/ceph.client.admin.keyring（admin 密钥）、/etc/ceph/ceph.pub（集群
公钥，orch 管理用）
账号 ps / 1（sudo 用户）；root 密码初始为 1（部署中开启 SSH 登录）
代理 示例 http://192.168.12.187:7897（国内访问 quay.io / GitHub 需要，按实际环境替
换）
1.2 三种存储接口对比
特性 RBD（块） CephFS（文件） RGW（对象）
访问接口 块设备 /dev/rbdX POSIX 文件系统 S3 / Swift API（HTTP）
多机共享 ❌ 单机独占 ✅ 多机共享读写 ✅
典型延迟 微秒级 毫秒级 毫秒 ~ 百毫秒
额外服务 无 需部署 MDS 需部署 RGW
客户端依赖 内核 rbd 模块 ceph-fuse / 内核 ceph 模块 任意 HTTP 客户端 / SDK
适用场景 虚拟机磁盘、数据库 共享目录、容器卷 图片、备份、静态文件、
API 存储


<!-- 第 4 页 -->

3
1.3 阅读指引
目标 阅读章节
从零搭建集群 第二部分（按 2.1 → 2.9 顺序执行）
扩容存储节点 第三部分
其他机器使用块存储 第四部分
多机共享文件目录 第五部分
应用通过 S3 API 存取数据 第六部分
Web 界面管理集群 / 文件 第七部分
排查故障 各部分末尾"常见问题"＋附录 A


<!-- 第 5 页 -->

4
第二部分 集群部署（副本模式）
总耗时：约 30~60 分钟（取决于网络）。执行角色区分：2.2、2.3 为三节点逐一执行；2.4
起除特殊标注外均在 ceph1 执行。
2.1 部署前准备
1. 添加数据盘：虚拟机平台给三台机器各加一块 ≥50GB 空白盘（不分区不格式化）。登录各节
点确认识别：
lsblk # 看到无分区的 sdb 即可
2. 网络互通：三台互 ping 通（任一节点执行）：
ping -c 2 192.168.12.176 && ping -c 2 192.168.12.90 && ping -c 2 192.168.12.169
2.2 系统初始化（三节点逐一执行）
以下命令在 ceph1、ceph2、ceph3 每台都执行（hostname 按对应节点修改）。
```bash
# 主机名（每台节点改成自己的名字）
hostnamectl set-hostname ceph1
# hosts 解析
cat >> /etc/hosts << 'HOSTS'
192.168.12.176 ceph1
192.168.12.90 ceph2
192.168.12.169 ceph3
HOSTS
# 禁用交换分区
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
# 容器运行时 + 基础工具
apt update && apt install -y podman lvm2 chrony curl
# 时间同步 + 关防火墙
systemctl enable --now chrony
ufw disable
✔ 验证：
podman --version # 输出版本号
chronyc tracking # System time 偏移毫秒级
free -m | grep Swap # total 为 0
2.3 开启 root SSH 登录（三节点逐一执行）
cephadm 需要 root 权限，Ubuntu 24.04 默认禁止 root SSH，需先用 ps 用户提权开启。
# 用 ps 用户执行（sudo 提权）
```


<!-- 第 6 页 -->

5
echo 'root:1' | sudo chpasswd
sudo sed -i 's/^#\?PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo sed -i 's/^#\?PasswordAuthentication.*/PasswordAuthentication yes/' \
/etc/ssh/sshd_config
sudo systemctl restart ssh
✔ 验证（ceph1 上测三台）：
ssh root@192.168.12.176 hostname # 输入密码 1，返回 ceph1
ssh root@192.168.12.90 hostname # 返回 ceph2
ssh root@192.168.12.169 hostname # 返回 ceph3
2.4 配置 SSH 免密（ceph1 执行）
```bash
# 生成密钥
ssh-keygen -t rsa -N '' -f /root/.ssh/id_rsa
# 安装 sshpass 并分发公钥到三台节点
apt install -y sshpass
sshpass -p '1' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.12.176
sshpass -p '1' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.12.90
sshpass -p '1' ssh-copy-id -o StrictHostKeyChecking=no root@192.168.12.169
✔ 验证：ssh root@ceph2 hostname 免密直接返回 ceph2。
2.5 安装 cephadm 并初始化集群（ceph1 执行）
⚠ 两个坑：① Squid 没有 noble（Ubuntu 24.04）包，必须用 jammy 源；② 新版 cephadm
是 zipapp，必须用包装脚本调用。
# 1. 添加 Ceph 源（trusted=yes 跳过 GPG，SSH 无 TTY 导不了密钥）
echo "deb [trusted=yes] https://download.ceph.com/debian-squid/ jammy main" \
> /etc/apt/sources.list.d/ceph.list
# 2. 安装
apt update && apt install -y cephadm
# 3. 创建包装脚本（必须！）
printf '#!/bin/bash\nexec python3 /usr/sbin/cephadm "$@"\n' > /usr/local/bin/cephadm
chmod +x /usr/local/bin/cephadm
# 4. 代理（写进 bashrc 让后续命令都生效）
cat >> /root/.bashrc << 'PROXY'
export http_proxy="http://192.168.12.187:7897"
export https_proxy="http://192.168.12.187:7897"
PROXY
source /root/.bashrc
# 5. bootstrap（拉取约 1.3GB 镜像，需 5~15 分钟）
cephadm bootstrap --mon-ip 192.168.12.176
```


<!-- 第 7 页 -->

6
✔ 验证与保存：cephadm version 输出版本号；bootstrap 完成后保存终端输出的 Dashboard
初始密码（忘记可用 2.10 节命令重置）。
```bash
# 检查初始状态（此时 /usr/bin/ceph 还没建，必须用 cephadm shell）
cephadm shell ceph -s
# 应有 1 MON、1 MGR、HEALTH_WARN、0 OSD
# 分发集群公钥到 ceph2/ceph3（ceph orch 管理节点用）
sshpass -p '1' ssh-copy-id -f -i /etc/ceph/ceph.pub \
-o StrictHostKeyChecking=no root@192.168.12.90
sshpass -p '1' ssh-copy-id -f -i /etc/ceph/ceph.pub \
-o StrictHostKeyChecking=no root@192.168.12.169
2.6 添加节点与扩展 MON（ceph1 执行）
# 添加节点
cephadm shell ceph orch host add ceph2 192.168.12.90
cephadm shell ceph orch host add ceph3 192.168.12.169
# 扩展 MON 到 3 个（自动部署到 ceph2/ceph3）
cephadm shell ceph orch apply mon 3
✔ 验证（等 1~2 分钟）：cephadm shell ceph orch host ls 显示 3 hosts；cephadm shell
ceph mon stat 显示 3 mons。
2.7 添加 OSD（ceph1 执行）
# 清空三台节点的数据盘（每台节点执行一次）
wipefs -a /dev/sdb
# 逐个添加 OSD
cephadm shell ceph orch daemon add osd ceph1:/dev/sdb
cephadm shell ceph orch daemon add osd ceph2:/dev/sdb
cephadm shell ceph orch daemon add osd ceph3:/dev/sdb
✔ 验证：cephadm shell ceph -s → osd: 3 osds: 3 up, 3 in，health 变 HEALTH_OK（新 OSD
会自动再平衡，等几分钟）。
若提示 3 pool(s) do not have an application enabled：cephadm shell ceph osd pool
application enable <池名> rbd
2.8 部署后命令行配置（ceph1 执行）
? ceph-common（含 ceph / rbd 命令）在 noble 上装不上（jammy 包依赖冲突），用容器
包装脚本替代。
# ceph 命令包装
cat > /usr/bin/ceph << 'CEPH'
```
#!/bin/bash
exec cephadm shell -- ceph "$@"
CEPH


<!-- 第 8 页 -->

7
chmod +x /usr/bin/ceph
```bash
# rbd 命令包装（map 系列需挂载宿主机 /sys /dev）
cat > /usr/bin/rbd << 'RBD'
```
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
```bash
# 加载 rbd 内核模块（仅需要 rbd map 时才需要）
modprobe rbd && echo 'rbd' > /etc/modules-load.d/rbd.conf
✔ 验证：ceph -s 正常输出集群状态（首次较慢，之后复用容器很快）。
2.9 RBD 功能测试（ceph1 执行）
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
✔ 验证：rbd ls mypool 输出 my-disk；df -h /mnt/ceph-rbd 显示约 4.9G 挂载成功。
2.10 Dashboard 访问（ceph1）
 地址：https://192.168.12.176:8443/（自签名证书，浏览器点"高级 → 继续前往"）
 账号：admin，初始密码为 bootstrap 输出的随机密码（后续已重置，当前密码见 7.2 节）
 密码重置（忘记或太弱时）：
echo "CephAdmin@2026" | cephadm shell -- ceph dashboard ac-user-set-password admin -i -
Ceph 19 强密码策略：至少 8 位，需含大写字母 + 数字，123456 会被拒。
```


<!-- 第 9 页 -->

8
2.11 部署常见问题速查
```bash
# 错误 解决
1 Permission denied (publickey,password) root SSH 未开启，回 2.3 节用 ps 用户改配
置
2 apt update 404 apt 源写成了 noble，应写 jammy main
3 cephadm: command not found 未创建 2.5 节包装脚本
4 gpg: cannot open '/dev/tty' 用 [trusted=yes] 跳过 GPG
5 rbd: sysfs write failed modprobe rbd
6 bootstrap 拉镜像极慢 配置代理后重跑
7 Dashboard 密码太弱 用 8 位以上含大小写 + 数字的密码
2.12 完全重置
# ceph1：清空集群（先记下 FSID）
ceph fsid
cephadm rm-cluster --fsid <FSID> --force
# 三台节点：清理 LVM 残留 + 数据盘
vgremove -f $(vgs --noheadings -o vg_name 2>/dev/null) 2>/dev/null
pvremove -f $(pvs --noheadings -o pv_name 2>/dev/null) 2>/dev/null
wipefs -a /dev/sdb
rm -f /etc/apt/sources.list.d/ceph.list
然后从 2.2 节重新开始。
```


<!-- 第 10 页 -->

9
第三部分 集群扩容：新增存储节点
使用方式：本章为通用模板，把 <NEW_HOSTNAME> 替换为新节点主机名、<NEW_IP> 替换为新节
点 IP（示例 ceph6 / 192.168.12.178）。
关键约定：① SSH 用密钥分发（Ubuntu 24.04 默认禁止 root 密码 SSH，sshpass 会失
败）；② ceph orch 依赖集群公钥 /etc/ceph/ceph.pub，必须分发到新节点。
适用场景：新节点仅完成 3 项准备（系统安装 + 静态 IP + 数据盘），其余配置按本章步骤
执行。
3.1 环境判断与节点规划
在 VMware / Proxmox 新建 Ubuntu 24.04 LTS 虚拟机，桥接模式与集群同网段。
资源 推荐配置
CPU / 内存 / 系统盘 4 核 / 6 GB / 50 GB
数据盘 1 块 × 50GB+ 独立裸盘（不分区不格式化，作新 OSD）
项目 值
主机名 <NEW_HOSTNAME>（示例 ceph6）
IP <NEW_IP>（示例 192.168.12.178，避开已用地址）
网关 / DNS 192.168.12.254 / 202.96.209.5、202.96.199.133
数据盘 /dev/sdb
选定 IP 前先 ping <NEW_IP> 确认无响应，并避开现有节点 IP。
3.2 前提条件
 ☑ 集群正常：ceph -s 显示 HEALTH_OK
 ☑ 新节点已装 Ubuntu 24.04 + 静态 IP，与集群互通
 ☑ 已添加空白数据盘 /dev/sdb
podman/chrony 安装、SSH 密钥分发行 无需提前准备，在下方操作步骤中完成。
3.3 操作步骤
步骤 1：系统初始化（新节点执行）
hostnamectl set-hostname <NEW_HOSTNAME>
cat >> /etc/hosts << 'HOSTS'
192.168.12.176 ceph1
192.168.12.90 ceph2
192.168.12.169 ceph3
192.168.12.157 ceph5
<NEW_IP> <NEW_HOSTNAME>


<!-- 第 11 页 -->

10
HOSTS
swapoff -a && sed -i '/swap/s/^/#/' /etc/fstab
apt update && apt install -y podman lvm2 chrony
systemctl enable --now chrony
ufw disable
✔ 验证：ping -c 1 ceph1 通；podman --version 输出 4.x；chronyc tracking 偏移毫秒级。
步骤 2：配置 SSH 免密（密钥分发）
⚠ Ubuntu 24.04 允许 root 密钥登录、禁止密码登录，不要用 sshpass 密码方式。
ceph.pub 必须分发（ceph orch 用它连接新节点）。
在 ceph1 获取密钥：
cat /etc/ceph/ceph.pub # 集群管理密钥（必须）
cat /root/.ssh/id_rsa.pub # ceph1 root 公钥（方便免密运维）
写入新节点 authorized_keys（方法 A / B 二选一）：
方法 A（ceph1 直推，前提 ceph1 已能连新节点）：
ssh-copy-id -i /etc/ceph/ceph.pub root@<NEW_IP>
ssh-copy-id root@<NEW_IP>
方法 B（新节点手动粘贴，最通用）：
sudo bash -c 'cat >> /root/.ssh/authorized_keys' # 粘贴上面两份公钥，Ctrl+D 结束
sudo chmod 700 /root/.ssh
sudo chmod 600 /root/.ssh/authorized_keys
✔ 验证（ceph1）：ssh root@<NEW_HOSTNAME> hostname 免密返回主机名。
步骤 3：加入集群（ceph1 执行）
ceph orch host add <NEW_HOSTNAME> <NEW_IP>
✔ 验证：ceph orch host ls 主机数量 +1；等 30~60 秒，ceph orch ps --host
<NEW_HOSTNAME> 有 exporter 等 daemon。
步骤 4：添加 OSD
```bash
# 新节点上先清空数据盘（在 <NEW_HOSTNAME> 执行）
wipefs -a /dev/sdb
# ceph1 上查看设备 → 添加 OSD
ceph orch device ls | grep <NEW_HOSTNAME> # AVAIL = Yes 表示可用
ceph orch daemon add osd <NEW_HOSTNAME>:/dev/sdb
✔ 验证：ceph -s | grep osd → OSD 数量 +1，up / in 状态相同；ceph osd tree 有新主机。
新 OSD 自动触发数据再平衡，后台进行，不影响现有业务。
```


<!-- 第 12 页 -->

11
步骤 5：监控数据再平衡（ceph1 执行）
添加 OSD 后 ceph -s 短暂显示 HEALTH_WARN（remapped / recovering）属正常现象，无需干
预：
ceph -w # 观察恢复进度，Ctrl+C 退出
ceph df # 查看容量变化
ceph pg stat # 所有 PG 回到 active+clean 即完成
再平衡通常 5~15 分钟完成，随后恢复 HEALTH_OK。
步骤 6：最终验证（ceph1 执行）
检查项 预期
ceph -s HEALTH_OK（再平衡后），mon 数量不变
OSD 数量 +1，up / in 状态相同
容量 原始 / 可用容量相应增加
MON 数量保持不变（3 个已满足高可用）。
3.4 常见问题
```bash
# 问题 解决
1 ceph orch host add 报 Failed to connect authorized_keys 缺 ceph.pub → 重做步骤 2，
确认免密
2 设备 AVAIL = No 磁盘有残留签名 → 新节点执行 wipefs -a
/dev/sdb
3 新 OSD 为 down 等待 30~60 秒；或 wipefs 后重新 add osd
4 HEALTH_WARN + remapped / recovering 再平衡中，正常，等自动完成
5 新节点 daemon 起不来 确认 podman 已装、chrony 误差 ≤50ms
3.5 注意事项
1. 数据盘必须是独立裸盘，不能是系统盘分区
2. 磁盘容量建议与现有 OSD 一致，否则容量分布不均
3. podman + chrony 必须预装，否则 daemon / OSD 无法启动
4. 再平衡期间性能受影响，生产建议维护窗口操作
5. 添加后无需重启集群，cephadm 自动管理
6. 移除节点：ceph orch host rm <NEW_HOSTNAME>（需先清空其 OSD）
7. 扩容后不可逆（删 OSD 会触发再平衡），生产操作需谨慎
8. IP 必须唯一：选 <NEW_IP> 前先 ping 确认无冲突
9. 加多台节点时，重复步骤 1~4 即可，全部添加完统一再平衡
```


<!-- 第 13 页 -->

12
第四部分 客户端接入：RBD 远程挂载
任务目标：在独立 Linux 客户端上安装 Ceph 工具，远程连接集群并使用 RBD 块存储。
4.1 环境与前提条件
资源 最低要求
CPU / 内存 / 系统盘 1 核 / 2 GB / 20 GB
网络 桥接模式，与集群同网段
规划：主机名 client，IP 192.168.12.237（建议值，须避开已用地址），网关 / DNS 见 1.1
节。
 □ 集群正常：ceph -s 显示 HEALTH_OK
 □ 客户端已装 Ubuntu 24.04，与集群网络互通（ping 192.168.12.176 通）
4.2 集群侧参考信息
项目 值
MON 地址 192.168.12.176 / .90 / .169（端口 6789）
FSID ceph fsid 获取
admin 密钥 cat /etc/ceph/ceph.client.admin.keyring 获取
4.3 操作步骤
步骤 1：客户端基础配置（客户端执行）
hostnamectl set-hostname client
cat >> /etc/hosts << 'HOSTS'
192.168.12.176 ceph1
192.168.12.90 ceph2
192.168.12.169 ceph3
192.168.12.237 client
HOSTS
```bash
# 静态 IP（根据实际规划修改网卡名与地址）
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
```


<!-- 第 14 页 -->

13
via: 192.168.12.254
nameservers:
addresses: [202.96.209.5, 202.96.199.133]
NETPLAN
netplan apply
✔ 验证：ping -c 2 192.168.12.176 通即可。
步骤 2：安装 ceph-common（客户端执行）
⚠ 客户端是 Ubuntu 24.04，同样用 jammy 源（Squid 无 noble 包），但 ceph-common 一
般能直接装上。
echo "deb [trusted=yes] https://download.ceph.com/debian-squid/ jammy main" \
> /etc/apt/sources.list.d/ceph.list
apt update && apt install -y ceph-common
✔ 验证：ceph --version 输出 ceph version 19.2.5 ... squid。
报依赖冲突时备选：apt install -y ceph-common --no-install-recommends；或参考 2.8 节
用容器包装脚本方案。
步骤 3：获取集群认证（ceph1 执行）
cat /etc/ceph/ceph.client.admin.keyring
```bash
# 记录输出中的 key 值，格式如：key = AQCDEf1234...==
步骤 4：配置客户端认证（客户端执行）
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
步骤 5：验证连接（客户端执行）
ceph -s
✔ 预期：显示 HEALTH_OK、3 MON、3 OSD。
失败时检查：ceph.conf 的 MON IP、网络（telnet 192.168.12.176 6789）、客户端防火墙
（ufw disable）。
```


<!-- 第 15 页 -->

14
步骤 6：远程使用 RBD（核心）
集群端准备测试镜像（ceph1 执行）：
ceph osd pool create client_pool 32 32
ceph osd pool application enable client_pool rbd
rbd create --size 5G client_pool/test_disk
客户端操作（客户端执行）：
```bash
# 查看远端镜像
rbd ls client_pool # 应输出 test_disk
# 映射（需内核模块）
modprobe rbd
rbd map client_pool/test_disk # 输出 /dev/rbd0
# 格式化 + 挂载（格式化仅首次）
mkfs.ext4 /dev/rbd0
mkdir -p /mnt/ceph-rbd && mount /dev/rbd0 /mnt/ceph-rbd
# 读写测试
echo "Ceph RBD remote mount test OK" > /mnt/ceph-rbd/test.txt
cat /mnt/ceph-rbd/test.txt
# 卸载
umount /mnt/ceph-rbd
rbd unmap client_pool/test_disk
✔ 验证：rbd showmapped 显示 /dev/rbd0；df -h /mnt/ceph-rbd 显示约 5G 容量。
步骤 7：开机自动挂载（可选）
cat > /usr/local/bin/mount-ceph-rbd.sh << 'SCRIPT'
```
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


<!-- 第 16 页 -->

15
WantedBy=multi-user.target
SERVICE
systemctl daemon-reload && systemctl enable ceph-rbd-mount.service
4.4 常见问题
```bash
# 问题 解决
1 ceph -s 连接超时 检查网络/防火墙，telnet 192.168.12.176 6789 测试
2 rbd map 报 sysfs write failed modprobe rbd
3 rbd map 报 Module rbd not found 安装 linux-modules-extra-$(uname -r)
4 mount: /dev/rbd0 is not a block device 先 rbd showmapped 确认映射成功
5 auth: incorrect key 检查 keyring 文件内容
```


<!-- 第 17 页 -->

16
第五部分 CephFS 文件存储
? 本集群实测结论先行：内核 7.0.0 + Ceph Squid 下内核客户端挂载报 libceph: socket
error on write，请使用 ceph-fuse。
5.1 CephFS 概述
5.1.1 架构与组件
CephFS 是 Ceph 提供的 POSIX 兼容分布式文件系统，底层复用同一 RADOS 存储池。多台客户
端可同时挂载同一个 CephFS 实现共享读写，行为类似 NFS 但无单点瓶颈。
客户端（ceph-fuse / 内核 mount -t ceph）
│ 文件请求
▼
MDS 元数据服务 ──→ cephfs_metadata 池（目录树/权限）
│
▼
cephfs_data 池（文件内容） ──→ RADOS（3 副本）
MDS 职责：维护目录树（inode / dentry）、与客户端协商权限（capability）、元数据缓存加
速。MDS 默认单线程，可启用 multiple active MDS 提升并发。
三种存储接口的横向对比见 1.2 节。
5.1.2 存储池布局
存储池 用途 副本 建议 PG
cephfs_metadata 文件名、目录结构、权限 3 32（元数据池不宜过大）
cephfs_data 文件实际内容 3 OSD 数 × 100 ÷ 副本数
5.2 前提条件与客户端选择
集群要求：
 □ Ceph 集群运行正常：ceph -s 显示 HEALTH_OK
 □ 集群至少 2 个 OSD
 □ MON 节点网络可达（客户端能访问 6789 / 3300 端口）
客户端选择：
客户端 依赖包 性能 本集群状态
ceph-fuse（推荐） ceph-fuse 中（用户态 FUSE） ? ✅ 已验证
内核客户端（备选） ceph-common 高（内核态） ? ❌ kernel 7.0 下不可用


<!-- 第 18 页 -->

17
本集群 CephFS 环境速查 ?：
项目 值
文件系统名 myfs
元数据池 / 数据池 cephfs_metadata（PG 32）/ cephfs_data（PG 128）
挂载点 /mnt/cephfs
MDS 部署 3 个（1 active + 2 standby，位于 ceph1/2/3）
5.3 部署 MDS 服务（ceph1 执行）
```bash
# 1. 创建存储池（PG：数据池 128，元数据池 32）
ceph osd pool create cephfs_metadata 32 32
ceph osd pool create cephfs_data 128 128
# 2. 创建文件系统
ceph fs new myfs cephfs_metadata cephfs_data
# 3. 部署 MDS（3 个：1 active + 2 standby）
ceph orch apply mds myfs --placement="3 ceph1 ceph2 ceph3"
✔ 验证：
ceph fs ls
# name: myfs, metadata pool: cephfs_metadata, data pools: [cephfs_data]
ceph mds stat
# myfs:1 {0=ceph1.xxxxxx=up:active} 2 up:standby
# 解读：myfs:1 = 1 个 active MDS；2 up:standby = 2 个备用（故障自动接管）
5.4 挂载方式选择
对比项 内核客户端 ceph-fuse
安装依赖 ceph-common ceph-fuse
性能 ★★★★★ ★★★☆☆
内核版本要求 需内核 ceph 模块 无
? 本集群状态 ❌ kernel 7.0 不可用 ✅ 已验证
5.5 ceph-fuse 挂载（推荐方案）
5.5.1 安装
# ? 必须从 Ubuntu 官方源安装（先移除 ceph 源避免依赖冲突）
rm -f /etc/apt/sources.list.d/ceph.list
apt update && apt install -y ceph-fuse
# 装完如需恢复 ceph 源
echo "deb [trusted=yes] https://download.ceph.com/debian-squid/ jammy main" \
```


<!-- 第 19 页 -->

18
> /etc/apt/sources.list.d/ceph.list
5.5.2 准备认证文件
在客户端创建 /etc/ceph/ceph.conf 和 ceph.client.admin.keyring（从集群 ceph1 复
制）：
mkdir -p /etc/ceph
```bash
# ceph.conf 内容：
# [global]
# fsid = <ceph fsid>
# mon host = 192.168.12.176, 192.168.12.90, 192.168.12.169
# ceph.client.admin.keyring：从 ceph1 的 /etc/ceph/ 直接拷贝即可
chmod 644 /etc/ceph/ceph.conf
chmod 600 /etc/ceph/ceph.client.admin.keyring
5.5.3 挂载与验证
mkdir -p /mnt/cephfs
ceph-fuse /mnt/cephfs # 默认读 keyring 认证
ceph-fuse /mnt/cephfs -m 192.168.12.176:6789 # 指定 MON
ceph-fuse /mnt/cephfs -n client.admin --keyring /etc/ceph/ceph.client.admin.keyring
# 多 MON：-m 192.168.12.176:6789,192.168.12.90:6789,192.168.12.169:6789
✔ 验证：
mount | grep cephfs
# ceph-fuse on /mnt/cephfs type fuse.ceph-fuse (rw,nosuid,nodev,relatime)
df -h /mnt/cephfs
5.5.4 常用参数与推荐组合
ceph-fuse /mnt/cephfs -m 192.168.12.176:6789 -n client.admin \
--keyring /etc/ceph/ceph.client.admin.keyring --noatime
常用参数 说明
-m 指定 MON 地址（默认自动发现）
-n 客户端 ID（默认 client.admin）
--keyring 指定 keyring 文件
--client_mountpoint 挂载子目录（如 /data/project-a）
-r / --noatime 只读 / 不更新访问时间
5.6 内核客户端挂载（其他兼容环境参考）
⚠ 本集群实测不兼容（socket error on write），本集群请直接用 5.5 节 ceph-fuse。
# 客户端安装
apt install -y ceph-common
# 认证文件（/etc/ceph/ceph.conf + ceph.client.admin.keyring，从集群 ceph1 复制）
mkdir -p /mnt/cephfs
mount -t ceph 192.168.12.176:6789:/ /mnt/cephfs -o name=admin
```


<!-- 第 20 页 -->

19
```bash
# 多 MON 容错：192.168.12.176,192.168.12.90,192.168.12.169:6789:/
# 免 keyring 文件：-o name=admin,secret=<密钥>
5.7 自动挂载与排查
systemd 自动挂载（推荐）：
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
⚠ 关键点：必须先加载 fuse 模块（modprobe fuse），否则 ceph-fuse 会报
Ignoring invalid max threads value 4294967295 > max (100000) 并退出，挂载点消失。
挂载消失的排查（如 File Browser 看不到文件，提示 "It feels lonely here..."）：
mount | grep cephfs # 1. 挂载是否还在
ps aux | grep ceph-fuse # 2. 进程是否存活
systemctl status cephfs-mount.service # 3. 服务状态
lsmod | grep fuse # 4. fuse 模块是否加载
journalctl -u cephfs-mount -n 30 # 5. 日志
modprobe fuse && systemctl restart cephfs-mount.service # 6. 修复
5.8 验证与日常操作
# 基本读写
mkdir -p /mnt/cephfs/data
echo "CephFS test file" > /mnt/cephfs/data/hello.txt
cat /mnt/cephfs/data/hello.txt
dd if=/dev/zero of=/mnt/cephfs/data/test.bin bs=1M count=100 # 大文件测试
```


<!-- 第 21 页 -->

20
```bash
# 多节点共享验证（ceph1 写入 → 客户端读取）
echo "shared data" > /mnt/cephfs/shared.txt
# 状态查看
mount | grep cephfs
ceph fs status myfs
ceph mds stat
ceph tell mds.0 client ls
5.9 高级功能
# 挂载子目录
ceph-fuse /mnt/project-a --client_mountpoint=/data/project-a
# 只读挂载
ceph-fuse /mnt/cephfs-ro -r
# CephFS 快照（需先启用）
ceph fs set myfs allow_new_snaps true
mkdir /mnt/cephfs/.snap/snap-20260727 # 创建快照
ls -la /mnt/cephfs/.snap/ # 查看
cp -a /mnt/cephfs/.snap/snap-20260727/* /mnt/cephfs/ # 回滚
# 配额（需安装 attr 包）
setfattr -n ceph.quota.max_bytes -v 107374182400 /mnt/cephfs/data # 100GiB
setfattr -n ceph.quota.max_files -v 100000 /mnt/cephfs/data
getfattr -n ceph.quota.max_bytes /mnt/cephfs/data
# 客户端细粒度授权（ceph1 执行）
ceph fs authorize myfs client.app1 / rw /data/app1 rw
# 输出的 key 给 app1 客户端挂载：-o name=app1,secret=<输出的 key>
5.10 故障排查
# 错误信息 原因 解决
1 no mds server is up MDS 未部署 / 宕机 ceph mds stat 确认有 active
2 Connection refused MON 端口不通 telnet 192.168.12.176 6789
3 auth: incorrect key keyring 密钥不匹配 ceph auth get client.admin 核
对
4 -13 Permission denied 认证失败 / 无权限 检查 keyring 内容与权限
5 no active rank 0 MDS 未就绪 等 30 秒重试
6 read-only file system 集群健康异常 ceph -s 修复降级 OSD / PG
7 ? socket error on
write（dmesg）
Kernel 7.0 与 Squid 协议不兼容 改用 ceph-fuse
```


<!-- 第 22 页 -->

21
8 挂载消失、fuse 报错退出 fuse 模块未加载 modprobe fuse + 配置
/etc/modules-load.d/fuse.conf
诊断思路：网络通？→ MDS 活？→ 认证对？→ 内核报错？（dmesg | grep ceph，有 socket error
就换 ceph-fuse）。
查看 MDS 日志与客户端会话管理：
cephadm enter mds.ceph1
tail -100 /var/log/ceph/ceph-mds.ceph1.log
exit
ceph tell mds.0 session ls
ceph tell mds.0 session evict id:<session_id> # 强制断开
5.11 卸载方法
```bash
# ceph-fuse / 内核客户端通用
umount /mnt/cephfs
# 失败时：lsof /mnt/cephfs 查占用 → umount -f /mnt/cephfs（强制）
# ceph-fuse 专用：fusermount -u /mnt/cephfs
# 清理 systemd 自动挂载
systemctl stop cephfs-mount.service
systemctl disable cephfs-mount.service
```


<!-- 第 23 页 -->

22
第六部分 RGW 对象存储（S3 兼容 API）
? 本章基于实际部署验证，RGW 服务名 myrgw，Squid 默认端口 80（非旧版 7480）。
6.1 概述
6.1.1 核心概念
RGW（RADOS Gateway）是 Ceph 的对象存储网关，对外暴露兼容 Amazon S3 的 RESTful
API。应用通过标准 HTTP/HTTPS 上传下载数据，无需安装 Ceph 专用客户端。
概念 S3 术语 说明
用户 User 访问身份，每个用户有独立 Access Key / Secret Key
存储桶 Bucket 对象的容器，类似文件夹但不可嵌套
对象 Object 存储基本单元 = 数据 + 元数据 + 唯一 Key
Access Key AK 用户标识（20 位字符）
Secret Key SK 访问密钥（40 位字符），用于签名请求
6.1.2 架构与存储位置
S3 客户端（s3cmd / AWS CLI / SDK）
│ HTTP S3 API
▼
RGW 网关（rgw.myrgw，默认监听 :80）
│ RADOS 原生协议
▼
RADOS 存储池（rgw_data 存对象内容 / rgw_meta、rgw_log 存元数据，3 副本分布所有节点）
RGW 只是协议翻译层：把 HTTP 的 S3 API 翻译成底层 RADOS 调用。文件最终切片、副本、
分布到多台节点的多块磁盘上（通过 S3 上传的文件实际存在 default.rgw.buckets.data 等池
里）。
6.2 前提条件与环境速查
 ☑ Ceph 集群运行正常：ceph -s 返回 HEALTH_OK
 ☑ 集群 OSD ≥ 3，容量充足
 ☑ 各节点网络互通
本集群 RGW 环境速查 ?：
项目 值
RGW 访问地址 http://192.168.12.176（Squid 默认端口 80）
RGW 服务名 myrgw
RGW 数据池 rgw_data（PG 128）/ rgw_meta、rgw_log（PG 32）


<!-- 第 24 页 -->

23
默认 S3 用户 testuser
6.3 部署 RGW 服务（ceph1 执行）
```bash
# 1. 创建存储池（可选，cephadm 也会自动建；数据池 128，元数据/日志池 32）
ceph osd pool create rgw_data 128 128
ceph osd pool create rgw_meta 32 32
ceph osd pool create rgw_log 32 32
# 2. 部署 RGW（三节点各一个实例）
ceph orch apply rgw myrgw --placement="3 ceph1 ceph2 ceph3"
✔ 验证：
ceph orch ps --daemon-type rgw
# 三行 rgw 实例，状态均 running
podman ps --format "{{.Names}} {{.Ports}}" | grep rgw
# 输出示例：ceph-a2173549...-rgw.myrgw.ceph1 0.0.0.0:80->8080/tcp
# ? Squid 默认端口是 80（不是旧版 7480）
6.4 创建 S3 用户与访问密钥（ceph1 执行）
? 本集群 ceph-common 未安装，radosgw-admin 需用 cephadm shell -- radosgw-admin 形
式执行（下同）。
# 创建用户（记录输出中的 access_key 和 secret_key）
cephadm shell -- radosgw-admin user create --uid=testuser --display-name="Test User"
# 查看用户
cephadm shell -- radosgw-admin user list
cephadm shell -- radosgw-admin user info --uid=testuser
# 创建多个用户（可选）
cephadm shell -- radosgw-admin user create --uid=app1 --display-name="Application 1"
# 为已有用户创建子密钥（可选）
cephadm shell -- radosgw-admin key create --uid=testuser --key-type=s3
输出示例（JSON）：
"keys": [ { "user": "testuser", "access_key": "ABCDEF...", "secret_key": "abcdef..." } ]
6.5 s3cmd 客户端
6.5.1 安装与配置
apt install -y s3cmd
# 交互配置，或直接写 ~/.s3cfg：
cat > ~/.s3cfg << 'CFG'
[default]
```


<!-- 第 25 页 -->

24
access_key = <你的 access_key>
secret_key = <你的 secret_key>
host_base = 192.168.12.176
host_bucket = 192.168.12.176
use_https = False
signature_v2 = False
CFG
? Squid 默认 80 端口，Endpoint 无需加端口号（非标准端口才需加 :端口）。
6.5.2 常用操作
s3cmd ls # 列出桶
s3cmd mb s3://my-bucket # 创建桶
s3cmd put ./file.txt s3://my-bucket/ # 上传
s3cmd ls s3://my-bucket/ # 列出对象
s3cmd get s3://my-bucket/file.txt ./ # 下载
s3cmd rm s3://my-bucket/file.txt # 删除
s3cmd rb s3://my-bucket # 删桶（需先清空）
s3cmd sync ./dir/ s3://my-bucket/ # 递归同步目录
s3cmd setacl s3://my-bucket/file.txt --acl-public # 设公开读
```bash
# 公开文件浏览器直接访问：http://192.168.12.176/my-bucket/file.txt
⚠ Ceph RGW 的 ACL 分 bucket 级 + object 级两层：桶设公开不会自动应用到对象，每个
文件需单独 --acl-public。
6.6 AWS CLI
pip install awscli
aws configure # 输入 AK/SK，region 留空
# 配置端点（写入 ~/.aws/config，后续免 --endpoint-url）
cat >> ~/.aws/config << 'CONF'
[default]
s3 =
endpoint_url = http://192.168.12.176
CONF
# 常用操作
aws --endpoint-url http://192.168.12.176 s3 ls
aws --endpoint-url http://192.168.12.176 s3 mb s3://aws-bucket
aws --endpoint-url http://192.168.12.176 s3 cp ./test.jpg s3://aws-bucket/
aws --endpoint-url http://192.168.12.176 s3 cp s3://aws-bucket/test.jpg ./
aws --endpoint-url http://192.168.12.176 s3 ls s3://aws-bucket/
aws --endpoint-url http://192.168.12.176 s3 presign \
s3://aws-bucket/test.jpg --expires-in 3600 # 预签名 URL
6.7 Python SDK（boto3）
import boto3
from botocore.client import Config
```


<!-- 第 26 页 -->

25
s3 = boto3.client(
"s3",
endpoint_url="http://192.168.12.176",
aws_access_key_id="<你的 access_key>",
aws_secret_access_key="<你的 secret_key>",
config=Config(signature_version="s3v4"),
region_name="",
)
s3.create_bucket(Bucket="my-python-bucket") # 创建桶
s3.put_object(Bucket="my-python-bucket", Key="hello.txt", Body=b"Hello RGW!") # 上传字符
串
s3.upload_file("/etc/hosts", "my-python-bucket", "hosts.txt") # 上传本地文件
s3.download_file("my-python-bucket", "hello.txt", "/tmp/download.txt") # 下载
for obj in s3.list_objects_v2(Bucket="my-python-bucket").get("Contents", []):
print(obj["Key"], obj["Size"]) # 列举
url = s3.generate_presigned_url("get_object",
Params={"Bucket": "my-python-bucket", "Key": "hello.txt"}, ExpiresIn=3600) # 预签名
s3.delete_object(Bucket="my-python-bucket", Key="hello.txt") # 删除
Node.js（aws-sdk）与 Java 的接入要点相同：endpoint 为 http://192.168.12.176、必
须开启 path-style（s3ForcePathStyle: true / withPathStyleAccessEnabled(true)）、签
名 v4。
6.8 高级配置
6.8.1 Nginx 反代 + HTTPS（生产推荐）
apt install -y nginx
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout /etc/nginx/ssl/rgw.key -out /etc/nginx/ssl/rgw.crt \
-subj "/CN=s3.ceph.local"
```bash
# /etc/nginx/sites-available/rgw
server {
listen 443 ssl;
server_name s3.ceph.local;
ssl_certificate /etc/nginx/ssl/rgw.crt;
ssl_certificate_key /etc/nginx/ssl/rgw.key;
location / {
proxy_pass http://127.0.0.1:80;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
}
}
ln -s /etc/nginx/sites-available/rgw /etc/nginx/sites-enabled/
nginx -s reload
```


<!-- 第 27 页 -->

26
6.8.2 配额、生命周期与 CORS
以下 radosgw-admin 命令在 ceph1 上需加 cephadm shell -- 前缀。
```bash
# 桶配额（10GB / 1000 对象）
cephadm shell -- radosgw-admin quota set --uid=testuser \
--quota-scope=bucket --max-size=10G
cephadm shell -- radosgw-admin quota set --uid=testuser \
--quota-scope=bucket --max-objects=1000
cephadm shell -- radosgw-admin quota enable --uid=testuser
# 用户配额（该用户所有桶总容量 100GB）
cephadm shell -- radosgw-admin quota set --uid=testuser \
--quota-scope=user --max-size=100G
# 生命周期：logs/ 前缀对象 30 天后过期
cat > lifecycle.json << 'JSON'
{ "Rules": [ { "ID": "expire-old-logs", "Status": "Enabled",
"Filter": { "Prefix": "logs/" }, "Expiration": { "Days": 30 } } ] }
JSON
aws --endpoint-url http://192.168.12.176 s3api put-bucket-lifecycle-configuration \
--bucket my-bucket --lifecycle-configuration file://lifecycle.json
# CORS（Web 前端跨域）
cat > cors.xml << 'XML'
<CORSConfiguration><CORSRule>
<AllowedOrigin>*</AllowedOrigin>
<AllowedMethod>GET</AllowedMethod>
<AllowedMethod>PUT</AllowedMethod>
<AllowedMethod>POST</AllowedMethod>
<AllowedHeader>*</AllowedHeader>
</CORSRule></CORSConfiguration>
XML
aws --endpoint-url http://192.168.12.176 s3api put-bucket-cors \
--bucket my-bucket --cors-configuration file://cors.xml
6.9 故障排查
# 错误信息 原因 解决
1 AccessDenied 密钥错误或无权限 检查 AK/SK；对象需单独设 public ACL
2 SignatureDoesNotMatch 客户端时间不同步 chronyc tracking 校正时间
3 BucketAlreadyExists 桶名已被占用 S3 桶名全局唯一，换名
4 EndpointConnectionErr
or
RGW 端口不可达 telnet 192.168.12.176 80
5 NoSuchBucket 桶不存在 / endpoint 错误 确认桶名与 endpoint URL
6 EntityTooLarge 单文件超 5GB 用分片上传（multipart）
7 InvalidAccessKeyId key 被禁用 / 删除 radosgw-admin user info 确认
8 curl 访问返回 403 需签名认证，不支持匿名 用 s3cmd/AWS CLI/SDK（403 属正常响应）
```


<!-- 第 28 页 -->

27
诊断流程：RGW 运行中？→ 端口通？→ 时间同步？→ 认证正确？
查看日志：
podman logs <rgw 容器名> 2>&1 | tail -50
cephadm enter --name rgw.myrgw.ceph1
tail -100 /var/log/ceph/ceph-client.rgw.myrgw.ceph1.log
exit
6.10 卸载与清理
```bash
# 1. 清空并删除桶
s3cmd del s3://my-bucket/*
s3cmd rb s3://my-bucket
# 或 aws --endpoint-url http://192.168.12.176 s3 rm s3://my-bucket --recursive
# 2. 删除 S3 用户
cephadm shell -- radosgw-admin user rm --uid=testuser
# 3. 删除 RGW 服务
ceph orch rm rgw.myrgw
# 4. 删除 RGW 存储池（确认无数据后）
ceph osd pool delete rgw_data rgw_data --yes-i-really-really-mean-it
ceph osd pool delete rgw_meta rgw_meta --yes-i-really-really-mean-it
ceph osd pool delete rgw_log rgw_log --yes-i-really-really-mean-it
```


<!-- 第 29 页 -->

28
第七部分 Web 管理界面
三个 Web 界面相互独立，可按需选用：Ceph Dashboard（集群管理，随 ceph-mgr 自动启
动）、File Browser（CephFS 文件管理）、Filestash（S3 对象存储管理）。除
Dashboard 外，后两者均部署在 ceph1 上。
7.1 总览与服务端口汇总
┌─ Ceph Dashboard (:8443) ── 集群状态 / 主机 / OSD / 存储池管理（ceph-mgr 自带）
│
├─ File Browser (:8080) ── 根目录 /mnt/cephfs ──→ CephFS（ceph-fuse 挂载）
│ 文件上传/下载/预览，多节点共享同一目录
│
└─ Filestash (:8334) ───── Endpoint :80 ──────→ RGW（S3 API）
S3 存储桶/对象管理，浏览器拖拽操作
服务 端口 状态 管理方式
Ceph Dashboard 8443 (HTTPS) ✅ 随 ceph-mgr 自动 ceph-mgr systemd
File Browser 8080 (HTTP) ✅ 运行 systemctl ... filebrowser
Filestash 8334 (HTTP) ✅ 运行 systemctl ... container-filestash
RGW（S3 API） 80 (HTTP) ✅ 运行 ceph orch
Nginx 反代 8444 (HTTP) ✅ 运行 systemctl ... nginx（cpolar 代理
Dashboard 用）
7.2 Ceph Dashboard（集群管理）
项目 值
访问地址 https://192.168.12.176:8443
用户名 admin
密码 Ps.123456 ?（初始为 bootstrap 随机密码，后重置；重置命令见 2.10 节）
⚠ HTTPS 自签名证书，浏览器首次访问需"高级 → 继续前往"。
功能：仪表盘（健康 / 容量 / IOPS）、主机、OSD、存储池、MON / MGR / MDS 监控、CephFS
状态、RGW 状态。
管理命令：
```bash
# 查看状态 / 端口
systemctl status ceph-a2173549-84bf-11f1-8f8e-000c29c6cc12@mgr.ceph1
ss -tlnp | grep 8443
# 重置密码
echo "新密码" | cephadm shell -- ceph dashboard ac-user-set-password admin -i -
```


<!-- 第 30 页 -->

29
7.3 File Browser（CephFS 文件管理）
轻量级 Web 文件管理器，根目录指向 CephFS 共享目录 /mnt/cephfs，多节点写入的文件立即
可见，支持浏览器上传 / 下载 / 删除 / 重命名、拖拽上传、文本 / PDF / 图片在线预览、分
享链接。
项目 值
访问地址 http://192.168.12.176:8080
用户名 / 密码 admin / Ps.123456
可执行文件 /usr/local/bin/filebrowser（v2.63.23）
数据库 / 日志 /etc/filebrowser.db / /var/log/filebrowser.log
⚠ 前提：/mnt/cephfs 必须已挂载（ceph-fuse）。若页面显示 "It feels lonely
here..." 且目录为空，按 5.7 节排查挂载。
7.3.1 下载并安装（ceph1 执行）
```bash
# 方法一：ceph1 直连 GitHub（网络可达时）
curl -fsSL -o /tmp/fb.tar.gz \
https://github.com/filebrowser/filebrowser/releases/latest/download/linux-amd64-
filebrowser.tar.gz
# 方法二：GitHub 被墙时，经 gh-proxy 代理下载（URL 较长，复制时勿断行）
curl -fsSL -o /tmp/fb.tar.gz \
"https://gh-
proxy.com/https://github.com/filebrowser/filebrowser/releases/download/v2.63.23/linux-
amd64-filebrowser.tar.gz"
# 方法三：本机下载后 SFTP 上传到 ceph1:/tmp/fb.tar.gz
# 解压安装
tar -xzf /tmp/fb.tar.gz -C /usr/local/bin/
chmod +x /usr/local/bin/filebrowser
/usr/local/bin/filebrowser version
# 应输出：File Browser v2.63.23/xxxx
7.3.2 初始化配置
# 初始化数据库（根目录 /mnt/cephfs，端口 8080）
filebrowser config init --root=/mnt/cephfs --address=0.0.0.0 --port=8080 \
--log=/var/log/filebrowser.log --database=/etc/filebrowser.db
# 放宽密码长度（默认要求 ≥12 位；如要短密码可设小）
filebrowser config set --minimumPasswordLength=6 --database=/etc/filebrowser.db
# 创建登录用户（用户名 admin，密码自定）
filebrowser users add admin Ps.123456 --perm.admin=true --database=/etc/filebrowser.db
```


<!-- 第 31 页 -->

30
7.3.3 注册 systemd 服务（开机自启）
cat > /etc/systemd/system/filebrowser.service << 'EOF'
[Unit]
Description=File Browser for CephFS
After=network.target
[Service]
ExecStart=/usr/local/bin/filebrowser -d /etc/filebrowser.db -r /mnt/cephfs
Restart=on-failure
RestartSec=5
[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable filebrowser.service
systemctl start filebrowser.service
7.3.4 验证
systemctl is-active filebrowser.service # 应输出 active
ss -tlnp | grep 8080 # 8080 端口监听
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8080/ # 200
✔ 浏览器访问：http://192.168.12.176:8080 ，应能看到 /mnt/cephfs 下的文件列表。
7.4 Filestash（S3 文件管理）
开源 S3 文件管理器，通过 Podman 容器部署，支持浏览器拖拽管理 RGW 存储桶和对象，功能最
完整的 S3 Web 界面。
项目 值
访问地址 http://192.168.12.176:8334
首次访问 /admin/setup 设置管理员密码
镜像 / 端口 docker.io/machines/filestash:latest / 8334:8334
持久化卷 filestash-data → /app/data（配置持久保存）
7.4.1 拉取镜像并启动（ceph1 执行）
```bash
# 拉取 Filestash 镜像（网络受限时可先 export http_proxy / https_proxy 代理地址）
podman pull docker.io/machines/filestash:latest
# 配置持久化卷（保存管理员密码和 S3 连接配置，容器重建不丢失）
podman volume create filestash-data
# 启动容器（端口 8334）
podman run -d --name filestash --restart=always \
-p 8334:8334 \
-v filestash-data:/app/data \
```


<!-- 第 32 页 -->

31
docker.io/machines/filestash:latest
podman ps --filter name=filestash --format "{{.Names}} {{.Status}} {{.Ports}}"
```bash
# 应输出：filestash Up X hours 0.0.0.0:8334->8334/tcp
7.4.2 注册 systemd 服务（推荐，更可靠的开机自启）
# 用 podman 自动生成标准 systemd 单元
podman generate systemd --name filestash --files --new
# 生成 container-filestash.service
cp container-filestash.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable container-filestash.service
# 切换由 systemd 管理（先停容器再通过服务启动）
podman stop filestash && podman rm filestash
systemctl start container-filestash.service
7.4.3 首次配置（浏览器操作）
访问 http://192.168.12.176:8334 ：
1. 设置管理员密码：自动跳转 /admin/setup，输入管理员密码（保存到持久化卷）
2. 添加 S3 存储：管理面板 → Storage → + Add new storage，类型选 S3：
字段 值
Endpoint http://192.168.12.176:80
Bucket test-bucket
Access Key S3 用户的 AK（本集群 testuser：3QQSL651T0L3UHILET23）
Secret Key S3 用户的 SK（本集群 testuser：VUVHSqbhrhXuBseKDSZTHZuRpdeVhCI0Bp4xRcwL）
Region 留空
Path style ✅ 勾选
3. 保存后即可在浏览器中浏览 / 上传 / 下载 / 搜索桶内对象。
7.4.4 验证
systemctl is-active container-filestash.service # active
ss -tlnp | grep 8334 # 8334 端口监听
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8334/ # 200（跳转 setup）
7.5 公网访问：cpolar 内网穿透
各服务已通过 cpolar 暴露公网（免费版 4 条隧道，Dashboard 占 2 条）。
⚠ 公网地址为动态分配，cpolar 每次重启 / 重连都会变化。获取当前地址：
grep "Tunnel established" /var/log/cpolar/access.log | tail -4
隧道配置（/usr/local/etc/cpolar/cpolar.yml）：
```


<!-- 第 33 页 -->

32
隧道名 协议 本地地址 对应服务
dashboard-tcp tcp 127.0.0.1:8443 Ceph Dashboard（TCP 备用）
dashboard-web http 127.0.0.1:8444 Ceph Dashboard（经 Nginx 反代）
filebrowser http 127.0.0.1:8080 File Browser
filestash http 127.0.0.1:8334 Filestash
Dashboard 因 HTTPS 自签名证书无法被 cpolar 直接代理，由 Nginx 监听 8444 反代到
8443（proxy_ssl_verify off）。
7.6 Web 界面常见问题
```bash
# 问题 解决
1 File Browser 下载 PDF 返回 ZIP 压缩包 ? v2.63.x 的下载接口用路径参数格式 /api/raw/<路
径>?inline=true，前端 SPA 默认使用正确格式；手动
测试勿用 ?path= 查询参数
2 File Browser 登录返回 401 JWT 过期：浏览器 F12 → Application → Local
Storage 清空该站点，重新登录
3 File Browser 页面空白 / 无文件 确认 CephFS 挂载存在、fuse 模块已加载，重启
cephfs-mount.service（见 5.7 节）
4 Filestash 访问跳转死循环 / 打不开 不要设置 APPLICATION_URL 环境变量（会拼出错误地
址），去掉后重启容器
5 Filestash 容器重建后配置丢失 必须挂载 -v filestash-data:/app/data 持久化卷
6 GitHub 下载 filebrowser 失败 用 gh-proxy 代理下载或本机下载后 SFTP 上传（见
7.3.1 节）
7 Filestash 连接 S3 报错 确认 Endpoint 带端口 80、勾选 Path style、AK/SK
正确
```


<!-- 第 34 页 -->

33
附录
附录 A 常用命令速查
类别 命令 说明
集群状态 ceph -s 健康状态、MON/OSD/PG 概览
集群状态 ceph -w 实时观察集群事件（Ctrl+C 退出）
集群状态 ceph df / ceph pg stat 容量与 PG 状态
集群状态 ceph osd tree OSD 拓扑（按主机分组）
服务编排 ceph orch host ls / host add / host rm 主机管理
服务编排 ceph orch ps / ps --daemon-type rgw daemon 运行状态
服务编排 ceph orch apply mon 3 / apply mds myfs ...
/ apply rgw myrgw ...
服务部署
服务编排 ceph orch daemon add osd <host>:/dev/sdb 添加 OSD
CephFS ceph fs ls / ceph mds stat / ceph fs status
myfs
文件系统与 MDS 状态
RGW cephadm shell -- radosgw-admin user list S3 用户管理（radosgw-admin 均加此前
缀）
容器 cephadm shell / cephadm enter --name
<daemon>
进入容器执行命令 / 调试
容器 podman ps / podman logs <容器名> 容器状态与日志
系统 systemctl status / start / stop <服务名> filebrowser / container-filestash
/ cephfs-mount 等服务
系统 journalctl -u <服务名> -n 30 查看服务日志
系统 chronyc tracking 时间同步偏移（排查认证类报错先看这
里）
附录 B 官方文档链接
主题 链接
Ceph 文档首页（Squid） https://docs.ceph.com/en/squid/
RBD 块存储 https://docs.ceph.com/en/squid/rbd/
CephFS 文件存储 https://docs.ceph.com/en/squid/cephfs/
RGW 对象存储 https://docs.ceph.com/en/squid/radosgw/
AWS S3 API 参考 https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html
