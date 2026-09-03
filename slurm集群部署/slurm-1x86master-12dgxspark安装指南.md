# SLURM 集群安装与部署教程（1×x86 Master + 12×DGX Spark）

> 基于《slurm-aarch64-dgxspark三节点安装指南》扩展，并汇总 **2026-08 实际部署** 中的配置、排错与验证步骤。  
> **SLURM 版本：24.11.5**（13 台必须一致）。  
> **拓扑：1 台 x86 Master + 12 台 DGX Spark（node1～node12）**。  
> **提交用户：`cds1_511`，全集群 uid/gid = 1010**（见 §2.3）。

---

## 0. 架构说明（x86 Master + aarch64 计算节点）

| 项目 | Master（x86） | Compute（DGX Spark / aarch64） |
|------|---------------|--------------------------------|
| 角色 | `slurmctld` + `slurmdbd` + `munge` + MariaDB | `slurmd` + `munge` |
| 二进制 | 在 **x86** 上编译安装 | 在 **各 aarch64 节点** 上分别编译安装 |
| 配置文件 | `slurm.conf` **内容一致**，从 Master 分发 | 与 Master 相同 |
| Munge | 生成 `munge.key`，同步到全部计算节点 | 使用同一份 `munge.key` |
| apt 镜像 | 标准 `ports.ubuntu.com` / 国内 x86 镜像 | **ubuntu-ports**（arm64） |

**结论：流程与三节点笔记相同，但 Master 与 Spark 需各自编译对应架构的二进制；不要混用 x86 的 `slurmd` 到 Spark 上。**

---

## 1. 规划

| 角色 | 主机名 | 架构 | 职责 | IP |
|------|--------|------|------|-----|
| Master | `master` | x86_64 | `slurmctld` + `slurmdbd` + `munge` + MariaDB | `10.128.12.52`（暂定） |
| Compute | `node1` | aarch64 | `slurmd` + `munge` | `10.128.12.40` |
| Compute | `node2` | aarch64 | `slurmd` + `munge` | `10.128.12.41` |
| Compute | `node3` | aarch64 | `slurmd` + `munge` | `10.128.12.42` |
| Compute | `node4` | aarch64 | `slurmd` + `munge` | `10.128.12.43` |
| Compute | `node5` | aarch64 | `slurmd` + `munge` | `10.128.12.44` |
| Compute | `node6` | aarch64 | `slurmd` + `munge` | `10.128.12.45` |
| Compute | `node7` | aarch64 | `slurmd` + `munge` | `10.128.12.46` |
| Compute | `node8` | aarch64 | `slurmd` + `munge` | `10.128.12.47` |
| Compute | `node9` | aarch64 | `slurmd` + `munge` | `10.128.12.48` |
| Compute | `node10` | aarch64 | `slurmd` + `munge` | `10.128.12.49` |
| Compute | `node11` | aarch64 | `slurmd` + `munge` | `10.128.12.50` |
| Compute | `node12` | aarch64 | `slurmd` + `munge` | `10.128.12.51` |

约定：

- **13 台** 使用 **同一 SLURM 版本**（24.11.5）、**同一份** `/etc/slurm/slurm.conf`
- **13 台** **同一份** `/etc/munge/munge.key`
- 时钟同步（chrony/ntp）
- `/etc/hosts` 或 DNS 能互相解析 `master`、`node1`～`node12`
- Master **只做控制节点**（不跑 `slurmd`）；计算仅在 node1～node12
- **记账（Accounting）** 在 Master 上：`MariaDB` + `slurmdbd`，记录作业 CPU/GPU/时长等，可用 `sacct` / `sacctmgr` 查询

网络建议（与 DGX Spark 实践一致）：

| 网段/接口 | 用途 |
|-----------|------|
| 10GbE RJ-45 | SSH、管理、NFS 挂载（若用） |
| ConnectX-7 QSFP（RoCE） | NCCL / 大流量（作业网络，可选单独网段） |

Slurm 调度只要求 **主机名互通**；管理网与 RoCE 网可分开规划 IP。

---

## 2. 全部节点：hosts 与时间

在 **Master + node1～node12** 上编辑 `/etc/hosts`：

```text
10.128.12.52  master
10.128.12.40  node1
10.128.12.41  node2
10.128.12.42  node3
10.128.12.43  node4
10.128.12.44  node5
10.128.12.45  node6
10.128.12.46  node7
10.128.12.47  node8
10.128.12.48  node9
10.128.12.49  node10
10.128.12.50  node11
10.128.12.51  node12
```

检查（任选一节点）：

```bash
hostname -s
for h in master node{1..12}; do ping -c1 -W1 $h && echo "$h OK" || echo "$h FAIL"; done
timedatectl   # 建议启用 NTP
```

### 2.1 SSH 免密（Master 批量运维必备）

Master 需能免密 SSH 到 12 台计算节点，用于分发配置、启服务、排错。

**推荐：Master 用 `root@node*`，或 `cds1_511@node*` + 免密 sudo。**

```bash
# 在 Master 上生成密钥（若尚无）
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519

# 方式 A：root 免密（部署阶段最省事）
for i in $(seq 1 12); do ssh-copy-id root@node$i; done

# 方式 B：cds1_511 免密（生产环境常用）
for i in $(seq 1 12); do ssh-copy-id cds1_511@node$i; done
```

验证：

```bash
ssh -o BatchMode=yes root@node1 hostname -s    # 应输出 node1
ssh -o BatchMode=yes cds1_511@node1 true       # 若用方式 B
```

> **常见坑**：`scp` 到 `/etc/munge/` 可能 Permission denied → 先 `scp` 到 `/tmp/munge.key` 或 `~/munge.key`，再 `ssh ... 'sudo cp ...'`。

### 2.2 下载 SLURM 源码（URL 注意 ASCII 连字符）

```bash
wget https://download.schedmd.com/slurm/slurm-24.11.5.tar.bz2
```

从 Word/PDF/聊天 **复制 URL 时**，中间的 `-` 可能变成 Unicode「智能连字符」（肉眼难辨），导致 **404**。务必 **手打** 版本号，或用上面 `wget` 命令。

目录页：https://download.schedmd.com/slurm/

### 2.3 集群用户与 UID 统一（重要）

Slurm 作业会在计算节点以提交用户的 **数值 UID** 执行 `chdir` 到 `$HOME`。若 Master 与节点 UID 不一致，会出现：

```text
slurmstepd: error: chdir(/home/cds1_511): No such file or directory
```

| 项 | 本集群约定 |
|----|------------|
| 提交用户 | `cds1_511` |
| UID / GID | **1010 / 1010**（全 13 台一致） |
| 选 1010 的原因 | node12 已有 `link@1001` 不能改；1000/1001 已被占用 |

**Master 上修改 UID**（用户若正在登录需先退出）：

```bash
# 若 cds1_511 正在使用，先结束进程
pkill -u cds1_511 || true
usermod -u 1010 cds1_511
groupmod -g 1010 cds1_511
chown -R cds1_511:cds1_511 /home/cds1_511
id cds1_511   # uid=1010 gid=1010
```

**12 台计算节点批量修改**（需 root SSH）：

```bash
for i in $(seq 1 12); do
  echo "===== node$i ====="
  ssh root@node$i "pkill -u cds1_511 2>/dev/null; usermod -u 1010 cds1_511; groupmod -g 1010 cds1_511; chown -R cds1_511:cds1_511 /home/cds1_511; id cds1_511"
done
```

**sacctmgr 登记提交用户**（slurmctld 启动后）：

```bash
export PATH=/usr/local/sbin:/usr/local/bin:$PATH
sacctmgr -i add user name=cds1_511 account=default cluster=spark
sacctmgr show user cds1_511 withassoc
```

---

## 3. 安装编译依赖

### 3.1 Master（x86）

```bash
sudo apt update
sudo NEEDRESTART_MODE=a apt install -y build-essential fakeroot \
  libmunge-dev libmysqlclient-dev libssl-dev libpam0g-dev \
  libjson-c-dev libhttp-parser-dev libyaml-dev libjwt-dev \
  libdbus-1-dev libnuma-dev libhdf5-dev munge mariadb-server
```

### 3.2 node1～node12（aarch64 / DGX Spark）

每台 Spark 执行（与三节点笔记相同）：

```bash
sudo apt update
sudo NEEDRESTART_MODE=a apt install -y build-essential fakeroot \
  libmunge-dev libmysqlclient-dev libssl-dev libpam0g-dev \
  libjson-c-dev libhttp-parser-dev libyaml-dev libjwt-dev \
  libdbus-1-dev libnuma-dev libhdf5-dev munge
```

> 批量执行示例（在 Master 上，需已配置 SSH 免密）：
>
> ```bash
> for i in $(seq 1 12); do
>   ssh root@node$i 'sudo NEEDRESTART_MODE=a apt install -y build-essential fakeroot libmunge-dev libmysqlclient-dev libssl-dev libpam0g-dev libjson-c-dev libhttp-parser-dev libyaml-dev libjwt-dev libdbus-1-dev libnuma-dev libhdf5-dev munge'
> done
> ```

---

## 4. 下载并编译 SLURM 24.11.5

### 4.1 仅在 Master（x86）上编译

```bash
VER=24.11.5
cd /tmp
wget https://download.schedmd.com/slurm/slurm-${VER}.tar.bz2
tar -xjf slurm-${VER}.tar.bz2
cd slurm-${VER}

./configure --prefix=/usr/local --sysconfdir=/etc/slurm --with-munge --with-mysql
make -j$(nproc)
sudo make install
sudo ldconfig

file /usr/local/sbin/slurmctld /usr/local/sbin/slurmdbd
# 应含 x86-64
```

### 4.2 在每台 node1～node12（aarch64）上分别编译

**每台 Spark 本地执行**（或 Ansible/循环 SSH，但必须在 aarch64 上编）：

```bash
VER=24.11.5
cd /tmp
wget https://download.schedmd.com/slurm/slurm-${VER}.tar.bz2
tar -xjf slurm-${VER}.tar.bz2
cd slurm-${VER}

./configure --prefix=/usr/local --sysconfdir=/etc/slurm --with-munge
make -j$(nproc)
sudo make install
sudo ldconfig

file /usr/local/sbin/slurmd
# 应含 ARM aarch64
```

批量编译示例（Master 上循环，编译在远端进行）：

```bash
VER=24.11.5
for i in $(seq 1 12); do
  echo "=== building on node$i ==="
  ssh root@node$i "bash -s" << 'REMOTE'
VER=24.11.5
cd /tmp
wget -q https://download.schedmd.com/slurm/slurm-${VER}.tar.bz2
tar -xjf slurm-${VER}.tar.bz2
cd slurm-${VER}
./configure --prefix=/usr/local --sysconfdir=/etc/slurm --with-munge
make -j$(nproc)
sudo make install
sudo ldconfig
file /usr/local/sbin/slurmd
REMOTE
done
```

---

## 5. Munge（Master 生成密钥，同步到 12 台计算节点）

### 5.1 仅在 Master 上生成

```bash
sudo mkdir -p /etc/munge
sudo dd if=/dev/urandom bs=1 count=1024 of=/etc/munge/munge.key
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key

sudo systemctl enable --now munge
munge -n | unmunge    # STATUS: Success
```

### 5.2 分发到 node1～node12

```bash
for i in $(seq 1 12); do
  scp /etc/munge/munge.key root@node$i:/etc/munge/munge.key
done
```

在 **每台计算节点** 上：

```bash
sudo chown munge:munge /etc/munge/munge.key
sudo chmod 400 /etc/munge/munge.key
sudo systemctl enable --now munge
munge -n | unmunge
```

批量：

```bash
for i in $(seq 1 12); do
  ssh root@node$i 'sudo chown munge:munge /etc/munge/munge.key; sudo chmod 400 /etc/munge/munge.key; sudo systemctl enable --now munge; munge -n | unmunge'
done
```

---

## 6. 目录与 `slurm.conf`（Master 编写，同步到全部计算节点）

### 6.1 创建目录（Master + 全部计算节点）

```bash
sudo mkdir -p /etc/slurm /var/spool/slurmd /var/spool/slurmctld /var/log/slurm
sudo chmod 755 /var/spool/slurmd /var/spool/slurmctld /var/log/slurm
```

计算节点只需 `slurmd` 相关目录（无 `slurmctld` 也可建全，无害）。

### 6.2 在 Master 上编写 `slurm.conf`

```bash
sudo tee /etc/slurm/slurm.conf << 'EOF'
ClusterName=spark
SlurmctldHost=master
MpiDefault=none
ProctrackType=proctrack/linuxproc
ReturnToService=1
SlurmctldPidFile=/var/run/slurmctld.pid
SlurmdPidFile=/var/run/slurmd.pid
SlurmdSpoolDir=/var/spool/slurmd
StateSaveLocation=/var/spool/slurmctld
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log
SlurmUser=root
SlurmdUser=root

# 记账（Accounting）
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=master
JobAcctGatherType=jobacct_gather/linux
JobAcctGatherFrequency=30

SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

# GPU 资源类型（DGX Spark 每节点 1× GB10）
GresTypes=gpu

# DGX Spark：20 CPU / ~121GiB RAM / 1× GPU（按 nvidia-smi、free -h 实测）
NodeName=node1  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node2  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node3  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node4  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node5  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node6  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node7  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node8  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node9  CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node10 CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node11 CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN
NodeName=node12 CPUs=20 RealMemory=122000 Gres=gpu:1 State=UNKNOWN

# 默认分区：12 节点
PartitionName=debug Nodes=node[1-12] Default=YES MaxTime=INFINITE State=UP
EOF
```

> **节点范围写法**：`node[1-12]` 在 Slurm 中表示 node1～node12；若版本不支持，改为逗号列表：  
> `Nodes=node1,node2,node3,node4,node5,node6,node7,node8,node9,node10,node11,node12`

### 6.3 同步到 node1～node12

```bash
for i in $(seq 1 12); do
  scp /etc/slurm/slurm.conf root@node$i:/etc/slurm/slurm.conf
done
```

### 6.4 GPU：`gres.conf`（各计算节点）

`slurm.conf` 已写 `Gres=gpu:1` 时，**每台 Spark** 需有 `/etc/slurm/gres.conf`：

```bash
sudo tee /etc/slurm/gres.conf << 'EOF'
AutoDetect=nvml
EOF
```

批量分发（Master 上）：

```bash
cat > /tmp/gres.conf << 'EOF'
AutoDetect=nvml
EOF
for i in $(seq 1 12); do
  scp /tmp/gres.conf root@node$i:/etc/slurm/gres.conf
  ssh root@node$i 'systemctl restart slurmd'
done
```

> 日志中 `_nvml_get_mem_freqs: Not Supported`、`nvidia_gb10` 命名警告 **可忽略**，不影响节点注册。  
> 不写 `Gres=gpu:1` 也能跑 GPU，但 Slurm **不会按 GPU 调度**，多作业可能抢同一块卡。

---

## 7. 记账（Accounting）：MariaDB + slurmdbd

> 仅 **Master** 需要。用于记录作业历史、账号/用户配额，支持 `sacct`、`sacctmgr`。

### 7.1 初始化 MariaDB

在 Master 上：

```bash
sudo systemctl enable --now mariadb

# 建库、建用户（密码请改成强密码）
sudo mysql << 'SQL'
CREATE DATABASE IF NOT EXISTS slurm_acct_db;
CREATE USER IF NOT EXISTS 'slurm'@'localhost' IDENTIFIED BY 'SlurmDBPass123!';
GRANT ALL PRIVILEGES ON slurm_acct_db.* TO 'slurm'@'localhost';
FLUSH PRIVILEGES;
SQL
```

### 7.2 编写 `slurmdbd.conf`

```bash
sudo tee /etc/slurm/slurmdbd.conf << 'EOF'
AuthType=auth/munge
DbdHost=master
DbdPort=6819
SlurmUser=root
DebugLevel=info
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurmdbd.pid

StorageType=accounting_storage/mysql
StorageHost=localhost
StorageUser=slurm
StoragePass=SlurmDBPass123!
StorageLoc=slurm_acct_db
EOF

sudo chmod 600 /etc/slurm/slurmdbd.conf
sudo chown root:root /etc/slurm/slurmdbd.conf
```

> `StoragePass` 须与 7.1 中数据库密码一致。

### 7.3 启动 `slurmdbd`（须先于 `slurmctld`）

```bash
sudo tee /etc/systemd/system/slurmdbd.service << 'EOF'
[Unit]
Description=Slurm DB daemon
After=network-online.target munge.service mariadb.service
Wants=network-online.target
Requires=munge.service

[Service]
Type=simple
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/usr/local/sbin/slurmdbd -D
ExecReload=/bin/kill -HUP $MAINPID
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now slurmdbd
sudo systemctl status slurmdbd --no-pager
```

### 7.4 初始化集群账号（sacctmgr）

在 **slurmctld 已启动** 后执行：

```bash
export PATH=/usr/local/sbin:/usr/local/bin:$PATH

# 注册集群（ClusterName 与 slurm.conf 一致）
sacctmgr -i add cluster name=spark

# 建账号、用户（示例；提交用户 cds1_511 必加）
sacctmgr -i add account name=default description="Default account" cluster=spark
sacctmgr -i add user name=root account=default cluster=spark
sacctmgr -i add user name=cds1_511 account=default cluster=spark

# 查看
sacctmgr show cluster
sacctmgr show account
sacctmgr show user
```

若希望 **未登记用户不能提交作业**，在 `slurm.conf` 增加：

```text
AccountingStorageEnforce=associations
```

改完后 `sudo systemctl restart slurmctld`。

---

## 8. systemd 服务

### 8.1 仅 Master：服务启动顺序

```text
munge → mariadb → slurmdbd → slurmctld
```

`slurmctld`：

```bash
sudo tee /etc/systemd/system/slurmctld.service << 'EOF'
[Unit]
Description=Slurm controller daemon
After=network-online.target munge.service slurmdbd.service
Wants=network-online.target
Requires=munge.service

[Service]
Type=simple
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/usr/local/sbin/slurmctld -D -s
ExecReload=/bin/kill -HUP $MAINPID
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now slurmctld
sudo systemctl status slurmctld --no-pager
```

### 8.2 node1～node12：`slurmd`

每台计算节点执行（可循环 SSH）：

```bash
sudo tee /etc/systemd/system/slurmd.service << 'EOF'
[Unit]
Description=Slurm node daemon
After=network-online.target munge.service
Wants=network-online.target
Requires=munge.service

[Service]
Type=simple
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/usr/local/sbin/slurmd -D -s
ExecReload=/bin/kill -HUP $MAINPID
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now slurmd
sudo systemctl status slurmd --no-pager
```

批量启动：

```bash
for i in $(seq 1 12); do
  ssh root@node$i 'sudo systemctl daemon-reload; sudo systemctl enable --now slurmd'
done
```

---

## 9. 验证（在 Master 上）

### 9.1 基础验证

```bash
export PATH=/usr/local/sbin:/usr/local/bin:$PATH

scontrol ping
sinfo -Nel
sinfo -p debug

# 12 节点各打一次主机名
srun -N12 -p debug hostname | sort

# 单节点试跑
srun -w node1 hostname
srun -w node12 hostname

# 以提交用户试跑（验证 UID / HOME）
su - cds1_511 -c 'srun -p debug -w node1 hostname'
su - cds1_511 -c 'srun -p debug -w node12 hostname'

# GPU 调度（可选）
su - cds1_511 -c 'srun -p debug --gres=gpu:1 nvidia-smi -L'

# 记账验证
srun -p debug sleep 3
sacct -X
sacctmgr show user withassoc
```

期望：

| 检查项 | 期望结果 |
|--------|----------|
| `scontrol ping` | Slurmctld 可达 |
| `sinfo -Nel` | node1～12 均为 **idle** |
| `srun -N12 hostname` | 12 行不同主机名 |
| `su - cds1_511 -c srun ...` | 无 chdir / Permission denied 警告 |
| `sacct` | 有作业历史 |
| `slurmdbd` | active |

### 9.2 一键健康检查脚本

仓库内脚本：`slurm-cluster-healthcheck.sh`（拷到 Master 执行）。

```bash
chmod +x slurm-cluster-healthcheck.sh
./slurm-cluster-healthcheck.sh
# 快速模式（跳过 srun）：QUICK=1 ./slurm-cluster-healthcheck.sh
```

脚本检查：Master 服务、munge 本地认证、munge.key md5 一致性、12 节点 SSH/服务/UID、`srun` 探针等。  
环境变量：`SUBMIT_USER=cds1_511`、`EXPECTED_UID=1010`、`NODE_COUNT=12`。

**全部通过示例**：`PASS=91  FAIL=0  WARN=0`，结论「集群健康检查通过」。

### 9.3 节点状态恢复

若某节点 `down` / `unknown`：

```bash
scontrol show node node3
scontrol update nodename=node3 state=resume
```

通用排错：

```bash
journalctl -u slurmctld -n 50 --no-pager          # Master
ssh root@node5 'journalctl -u slurmd -n 50 --no-pager'
tail -50 /var/log/slurm/slurmctld.log
ssh root@node5 'tail -30 /var/log/slurm/slurmd.log'
```

---

## 10. 常见问题与排错（本次部署实录）

### 10.1 速查表

| 现象 | 原因 / 处理 |
|------|-------------|
| 节点 `unknown*` | `slurmd` 未跑、无 unit 文件、munge 失败、主机名与 `NodeName=` 不一致 |
| `Unit file slurmd.service does not exist` | 手动写 systemd unit（§8.2），再 `daemon-reload && enable --now` |
| `Invalid credential` / `Protocol authentication error` | **13 台 `munge.key` 不一致** → 从 Master 重分发并 `restart munge slurmd` |
| `slurmstepd: chdir(/home/cds1_511)` 失败 | Master 与节点 **UID 不一致** → 全集群统一为 1010（§2.3） |
| `scp` / `sudo` 要密码 | 配置 root 免密或 `cds1_511` 免密 sudo；文件先拷 `/tmp` 再 `sudo cp` |
| Master 上 `slurmd` 架构错误 | Master 只装 `slurmctld`/`slurmdbd`，不要拷 x86 `slurmd` 到 Spark |
| Spark 上 exec format error | 用了 x86 二进制 → 在 aarch64 上重新编译 |
| `sacct` 无数据 | `slurmdbd` 未启；MariaDB 未建库；启动顺序不对 |
| slurmctld accounting 报错 | 先 `munge → mariadb → slurmdbd → slurmctld` |
| 下载 404 | URL 中 `-` 须为 **ASCII** 连字符（§2.2） |
| 健康检查 munge FAIL 但 srun 成功 | 旧版脚本误判；用最新 `slurm-cluster-healthcheck.sh` |

### 10.2 节点 unknown：逐步排查

```bash
# 1. 批量看 slurmd 是否在跑
for i in $(seq 1 12); do
  echo "===== node$i ====="
  ssh root@node$i 'systemctl is-active munge slurmd; test -f /etc/systemd/system/slurmd.service && echo unit:OK || echo unit:MISSING'
done

# 2. 主机名必须与 slurm.conf 一致
for i in $(seq 1 12); do ssh root@node$i "echo -n node$i: ; hostname -s"; done

# 3. munge.key md5 必须一致
md5sum /etc/munge/munge.key
ssh root@node1 'md5sum /etc/munge/munge.key'
```

### 10.3 重新同步 munge.key（整段可复用）

```bash
for i in $(seq 1 12); do
  echo "===== node$i ====="
  scp /etc/munge/munge.key root@node$i:/tmp/munge.key
  ssh root@node$i 'cp /tmp/munge.key /etc/munge/munge.key && chown munge:munge /etc/munge/munge.key && chmod 400 /etc/munge/munge.key && rm -f /tmp/munge.key && systemctl restart munge slurmd && munge -n | unmunge | head -1'
done
sudo systemctl restart munge slurmctld
sinfo -Nel
```

### 10.4 批量创建并启动 slurmd.service

若编译安装后 **没有** 自带 unit 文件：

```bash
cat > /tmp/slurmd.service << 'EOF'
[Unit]
Description=Slurm node daemon
After=network-online.target munge.service
Wants=network-online.target
Requires=munge.service

[Service]
Type=simple
Environment=PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin
ExecStart=/usr/local/sbin/slurmd -D -s
LimitNOFILE=65536
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

for i in $(seq 1 12); do
  scp /tmp/slurmd.service root@node$i:/etc/systemd/system/slurmd.service
  ssh root@node$i 'systemctl daemon-reload && systemctl enable --now slurmd && systemctl is-active munge slurmd'
done
```

### 10.5 Master 服务启动顺序（牢记）

```text
munge → mariadb → slurmdbd → slurmctld
```

计算节点：

```text
munge → slurmd
```

---

## 11. 步骤清单（速查）

| 步骤 | Master（x86） | node1～node12（aarch64） |
|------|---------------|---------------------------|
| 1 | `/etc/hosts` + NTP + SSH 免密 | 同左 |
| 2 | 统一 `cds1_511` uid=1010 | 同左 |
| 3 | apt 依赖 + mariadb-server | apt 依赖（arm64） |
| 4 | 编译 SLURM（`--with-mysql`） | 每台本地编译（`--with-munge`） |
| 5 | 生成 `munge.key`，启动 munge | 接收 key，启动 munge |
| 6 | 建库、`slurmdbd.conf`、启 `slurmdbd` | — |
| 7 | 编写 `slurm.conf`（CPU/内存/GPU/Accounting） | 接收 `slurm.conf` + `gres.conf` |
| 8 | 写 systemd unit，启 `slurmctld` | 写 unit，启 `slurmd` |
| 9 | `sacctmgr` 建 cluster/account/user | — |
| 10 | `sinfo` / `srun` / `sacct` / 健康检查脚本 | — |

---

## 12. 部署完成标准（验收）

满足以下即可认为集群 **可用**：

1. `scontrol ping` → UP  
2. `sinfo -Nel` → 12 节点 **idle**，MEMORY=122000，Gres 可见 gpu  
3. `su - cds1_511 -c 'srun -p debug -w node1 hostname'` → `node1`，无 chdir 警告  
4. `su - cds1_511 -c 'srun -p debug -w node12 hostname'` → `node12`  
5. `sacct -X` 有历史记录；`sacctmgr show user cds1_511` 有登记  
6. `./slurm-cluster-healthcheck.sh` → **FAIL=0**

---

## 13. 参考与附件

| 文件 | 说明 |
|------|------|
| `slurm-1x86master-12dgxspark安装指南.md` | 本文（安装 + 部署 + 排错） |
| `slurm-aarch64-dgxspark三节点安装指南.md` | 三节点前置笔记 |
| `slurm-cluster-healthcheck.sh` | 集群健康检查脚本 |

外部链接：

- 官方下载：https://download.schedmd.com/slurm/  
- 快速管理：https://slurm.schedmd.com/quickstart_admin.html  

硬件环境：Ubuntu aarch64、DGX Spark / GB10（每节点 1 GPU）、ConnectX-7 RoCE（可选作业网，Slurm 调度仅需管理网主机名互通）。
