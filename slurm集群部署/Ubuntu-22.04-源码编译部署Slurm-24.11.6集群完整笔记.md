# Ubuntu 22.04 源码编译部署 Slurm 24.11.6 集群完整笔记

> 适配环境：双节点集群，master 兼任控制+计算节点，node1 为纯计算节点，带 slurmdbd 记账服务

---

## 一、环境与节点规划

### 1.1 基础环境

| 项目 | 说明 |
| --- | --- |
| 操作系统 | Ubuntu 22.04 LTS |
| Slurm 版本 | 24.11.6（源码编译安装） |
| 认证方式 | MUNGE |
| 资源隔离 | cgroup v2 |
| 记账服务 | slurmdbd + MariaDB |

### 1.2 节点规划

| IP 地址 | 主机名 | 角色 | CPU | 内存 |
| --- | --- | --- | --- | --- |
| 192.168.12.25 | master | 控制节点(slurmctld) + 计算节点(slurmd) + 记账节点(slurmdbd) | 8核 | 3868MB |
| 192.168.12.80 | node1 | 纯计算节点(slurmd) | 8核 | 3868MB |

---

## 二、基础环境配置（所有节点都执行）

### 2.1 设置主机名

master 节点执行：

```bash
sudo hostnamectl set-hostname master
```

node1 节点执行：

```bash
sudo hostnamectl set-hostname node1
```

### 2.2 配置 hosts 解析

两台机器都追加以下内容（保留原有内容）：

```bash
sudo tee -a /etc/hosts <<EOF
192.168.12.25 master
192.168.12.80 node1
EOF
```

配置完成后互相 ping 测试连通性。

### 2.3 关闭防火墙（虚拟机测试环境）

```bash
sudo systemctl stop ufw
sudo systemctl disable ufw
sudo iptables -F
```

### 2.4 时间同步（必须配置，否则 MUNGE 认证失败）

```bash
sudo apt update
sudo apt install -y chrony
sudo systemctl enable --now chrony
```

### 2.5 修改系统资源限制

```bash
sudo tee -a /etc/security/limits.conf <<EOF
* soft nofile 65535
* hard nofile 65535
* soft nproc 65535
* hard nproc 65535
EOF
```

---

## 三、创建统一系统用户（所有节点都执行，UID/GID 必须完全一致）

> MUNGE 和 Slurm 要求集群内所有节点用户 ID、组 ID 完全相同，否则会出现权限和认证问题。

```bash
# 创建 munge 用户（固定 UID 2000）
sudo groupadd -g 2000 munge
sudo useradd -m -c "MUNGE User" -d /var/lib/munge -u 2000 -g munge -s /sbin/nologin munge

# 创建 slurm 用户（固定 UID 64031）
sudo groupadd -r slurm -g 64031
sudo useradd -r -g slurm -u 64031 -s /bin/false slurm
```

---

## 四、配置 SSH 免密登录（仅 master 节点执行）

### 4.1 开启 root 密码登录（所有节点都执行）

```bash
sudo sed -i 's/^#*PermitRootLogin.*/PermitRootLogin yes/' /etc/ssh/sshd_config
sudo sed -i 's/^#*PasswordAuthentication.*/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo systemctl restart sshd
# 设置 root 密码
sudo passwd root
```

### 4.2 master 生成密钥并分发

```bash
# 生成密钥，一路回车即可
ssh-keygen -t rsa -b 4096

# 分发到本机和计算节点
ssh-copy-id -i ~/.ssh/id_rsa.pub root@master
ssh-copy-id -i ~/.ssh/id_rsa.pub root@node1

# 测试免密是否生效
ssh root@node1 hostname
```

---

## 五、安装配置 MUNGE 认证服务（所有节点执行）

> MUNGE 是 Slurm 的身份认证组件，所有节点必须使用同一份密钥文件。

### 5.1 安装 MUNGE

```bash
sudo apt install -y munge libmunge-dev libmunge2
```

### 5.2 仅在 master 生成密钥

```bash
sudo dd if=/dev/urandom bs=1 count=1024 of=/etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
```

### 5.3 同步密钥到 node1

```bash
sudo scp /etc/munge/munge.key root@node1:/etc/munge/
ssh root@node1 "chown munge:munge /etc/munge/munge.key && chmod 400 /etc/munge/munge.key"
```

### 5.4 所有节点启动服务并验证

```bash
sudo systemctl enable munge
sudo systemctl restart munge
# 确认状态为 active (running)
sudo systemctl status munge
```

master 上测试认证：

```bash
munge -n | unmunge
```

无报错即表示 MUNGE 正常工作。

---

## 六、编译依赖与目录准备（所有节点执行）

### 6.1 安装编译依赖包

```bash
sudo apt update
sudo apt install -y build-essential git wget \
libmunge-dev libmunge2 munge \
libssl-dev libpam0g-dev libnuma-dev \
libhwloc-dev librrd-dev libncurses-dev \
libtool autoconf automake pkg-config \
libdbus-1-dev libibumad-dev libibverbs-dev \
libfreeipmi-dev libjson-c-dev libhttp-parser-dev \
libjwt-dev liblz4-dev libreadline-dev \
libcurl4-openssl-dev libhdf5-dev \
libmysqlclient-dev libevent-dev \
man2html html2text python3 python3-pip
```

### 6.2 创建 Slurm 工作目录

```bash
sudo mkdir -p /etc/slurm
sudo mkdir -p /var/spool/slurm/ctld
sudo mkdir -p /var/spool/slurm/d
sudo mkdir -p /var/log/slurm
sudo mkdir -p /var/run/slurm

sudo chown -R slurm:slurm /var/spool/slurm /var/log/slurm /var/run/slurm
sudo chmod 755 /var/spool/slurm
```

---

## 七、源码编译安装 Slurm 24.11.6（所有节点执行）

### 7.1 下载并解压源码

```bash
cd /usr/local/src
sudo wget https://download.schedmd.com/slurm/slurm-24.11.6.tar.bz2
sudo tar -xjf slurm-24.11.6.tar.bz2
cd slurm-24.11.6
```

### 7.2 配置编译参数

```bash
sudo ./configure \
--prefix=/usr/local \
--sysconfdir=/etc/slurm \
--with-munge \
--with-pam \
--with-hwloc \
--with-numa \
--with-ssl \
--with-json \
--with-jwt \
--enable-slurmrestd \
--disable-debug \
--disable-dependency-tracking
```

执行完成后确认输出末尾无 error，MUNGE、HWLOC 等核心组件显示为 yes。

### 7.3 编译与安装

```bash
# 并发编译，根据机器性能调整 -j 后数值
make -j$(nproc)
sudo make install
```

### 7.4 后续配置

```bash
# 更新动态链接库缓存
sudo ldconfig /usr/local/lib

# 复制 systemd 服务文件
sudo cp etc/slurmctld.service /etc/systemd/system/
sudo cp etc/slurmd.service /etc/systemd/system/
sudo cp etc/slurmdbd.service /etc/systemd/system/
sudo systemctl daemon-reload
```

---

## 八、Slurm 核心配置文件

### 8.1 确认硬件参数（master 执行）

> 配置文件中的 CPU 和内存必须填真实值，否则节点会启动失败。

```bash
# 查看 CPU 核数
lscpu | grep "^CPU(s):"

# 查看总内存（单位 MB）
free -m | grep Mem | awk '{print $2}'
```

### 8.2 生成 slurm.conf（仅 master 执行）

> 已适配 8核/3800MB 内存（留少量系统余量），无 GPU 环境；记账配置先注释，等部署完 slurmdbd 再开启。

```bash
sudo tee /etc/slurm/slurm.conf <<'EOF'
ClusterName=mycluster
SlurmctldHost=master
SlurmUser=slurm
SlurmdUser=root
AuthType=auth/munge
CryptoType=crypto/munge

# 记账功能（部署 slurmdbd 后再取消注释）
# AccountingStorageType=accounting_storage/slurmdbd
# AccountingStorageHost=localhost

MpiDefault=none
ProctrackType=proctrack/cgroup
ReturnToService=2
StateSaveLocation=/var/spool/slurm/ctld
SlurmdSpoolDir=/var/spool/slurm/d
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
SlurmctldDebug=info
SlurmdDebug=info
SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory
TaskPlugin=task/affinity,task/cgroup
JobAcctGatherType=jobacct_gather/cgroup
JobAcctGatherFrequency=task=30

PartitionName=debug Nodes=ALL Default=YES MaxTime=INFINITE State=UP

# 节点配置
NodeName=master NodeAddr=192.168.12.25 CPUs=8 RealMemory=3800 State=UNKNOWN
NodeName=node1 NodeAddr=192.168.12.80 CPUs=8 RealMemory=3800 State=UNKNOWN
EOF
```

### 8.3 生成 cgroup.conf（仅 master 执行）

> 适配 Ubuntu 22.04 默认的 cgroup v2，已移除 24.11 版本废弃的 CgroupAutomount 参数。

```bash
sudo tee /etc/slurm/cgroup.conf <<'EOF'
CgroupPlugin=cgroup/v2
CgroupMountpoint=/sys/fs/cgroup
ConstrainCores=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=no
ConstrainDevices=no
AllowedRAMSpace=98
EOF
```

### 8.4 同步配置到 node1

```bash
sudo scp /etc/slurm/slurm.conf root@node1:/etc/slurm/
sudo scp /etc/slurm/cgroup.conf root@node1:/etc/slurm/
```

---

## 九、启动集群服务并验证

### 9.1 启动服务

master 节点：

```bash
# 启动控制服务
sudo systemctl enable slurmctld
sudo systemctl start slurmctld

# master 同时作为计算节点，启动 slurmd
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

node1 节点：

```bash
sudo systemctl enable slurmd
sudo systemctl start slurmd
```

### 9.2 集群状态验证（master 执行）

```bash
# 查看节点和分区状态，节点应为 idle
sinfo

# 查看节点详细信息
scontrol show nodes

# 测试控制节点通信
scontrol ping

# 提交跨节点测试作业
srun -N2 -n2 hostname

# 查看作业队列
squeue
```

节点状态异常恢复命令：

```bash
sudo scontrol update nodename=master state=resume
sudo scontrol update nodename=node1 state=resume
```

---

## 十、部署 slurmdbd 记账服务（仅 master 执行）

### 10.1 安装 MariaDB 数据库

```bash
sudo apt install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb
```

### 10.2 创建数据库与用户

```bash
sudo mysql -u root
```

进入 MySQL 命令行后执行：

```sql
CREATE DATABASE slurm_acct_db;
CREATE USER 'slurm'@'localhost' IDENTIFIED BY 'Slurm@123456';
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
exit;
```

### 10.3 配置 slurmdbd.conf

```bash
sudo tee /etc/slurm/slurmdbd.conf <<'EOF'
AuthType=auth/munge
DbdHost=localhost
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageUser=slurm
StoragePass=Slurm@123456
StorageLoc=slurm_acct_db
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurm/slurmdbd.pid
SlurmUser=slurm
EOF
```

### 10.4 修正权限并启动服务

```bash
sudo chown slurm:slurm /etc/slurm/slurmdbd.conf
sudo chmod 600 /etc/slurm/slurmdbd.conf
sudo chown -R slurm:slurm /var/log/slurm /var/run/slurm

sudo systemctl daemon-reload
sudo systemctl enable --now slurmdbd
# 确认服务正常运行
sudo systemctl status slurmdbd
```

### 10.5 开启 slurmctld 记账功能

> ⚠ 注意：开启前必须先停止 slurmctld 并清空本地状态文件，避免 Cluster ID 不匹配导致启动失败。

```bash
# 停止 slurmctld
sudo systemctl stop slurmctld

# 清空本地状态文件
sudo rm -rf /var/spool/slurm/ctld/*

# 取消 slurm.conf 中记账配置的注释
sudo sed -i 's/^# AccountingStorageType/AccountingStorageType/' /etc/slurm/slurm.conf
sudo sed -i 's/^# AccountingStorageHost/AccountingStorageHost/' /etc/slurm/slurm.conf

# 同步配置到 node1（可选，需要 node1 支持 sacct 查询时同步）
sudo scp /etc/slurm/slurm.conf root@node1:/etc/slurm/

# 重启 slurmctld
sudo systemctl start slurmctld
```

### 10.6 初始化集群并验证

```bash
# 添加集群（提示已存在属于正常现象）
sacctmgr add cluster mycluster

# 查看集群列表
sacctmgr show cluster

# 提交测试作业后查询记账记录
srun hostname
sacct
```

能查到作业记录即为记账功能部署完成。

---

## 十一、常用运维命令

### 11.1 节点与分区管理

```bash
sinfo # 查看分区和节点状态
scontrol show nodes # 查看节点详细信息
scontrol show part # 查看分区详细配置
scontrol update nodename=node1 state=resume # 恢复节点状态
scontrol update nodename=node1 state=drain reason=维护 # 排空节点
```

### 11.2 作业提交与管理

```bash
srun -N2 -n4 hostname # 交互式提交作业
sbatch job.sh # 批量提交作业脚本
squeue # 查看作业队列
scancel 123 # 取消指定ID的作业
scontrol show job 123 # 查看作业详细信息
```

### 11.3 记账查询

```bash
sacct # 查看历史作业
sacct -j 123 --format=JobID,JobName,State,Elapsed,AllocCPUS,MaxRSS # 查看指定作业详情
sacctmgr show cluster # 查看集群记账信息
sacctmgr show account # 查看账户信息
```

### 11.4 服务管理

```bash
# master 节点
systemctl status slurmctld
systemctl status slurmd
systemctl status slurmdbd

# 计算节点
systemctl status slurmd
```

---

## 十二、常见问题排查

### 12.1 slurmctld 启动失败：CLUSTER ID MISMATCH

- **原因**：先启动过无记账的 slurmctld，本地生成了 ClusterID，开启记账后与数据库 ID 冲突。
- **解决**：停止 slurmctld，执行 `sudo rm -rf /var/spool/slurm/ctld/*` 清空状态文件，再重启服务。

### 12.2 cgroup.conf 警告：CgroupAutomount is defunct

- **原因**：Slurm 24.11 版本已废弃该参数。
- **解决**：从 cgroup.conf 中删除 `CgroupAutomount=yes` 行。

### 12.3 节点状态为 down/drain

- **优先检查**：节点间时间是否同步、munge 密钥是否一致、slurm.conf 中 CPU/内存数值是否超过实际值、主机名解析是否正确。
- **手动恢复**：`sudo scontrol update nodename=节点名 state=resume`

### 12.4 MUNGE 认证失败

- 检查所有节点时间差不超过 5 秒
- 检查 `/etc/munge/munge.key` 权限为 400，所有者为 `munge:munge`
- 确认所有节点使用同一份密钥文件

### 12.5 计算节点执行 sacct 提示 accounting storage is disabled

- **原因**：计算节点 slurm.conf 未配置记账地址，且 slurmdbd 默认只监听本地。
- **解决**：管理操作统一在 master 执行；如需计算节点查询，修改 slurmdbd 监听 0.0.0.0 并同步完整 slurm.conf 到计算节点。

---

*本笔记来源于 PDF 文档转换，已保留原文档章节结构与完整命令。*