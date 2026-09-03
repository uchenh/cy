# Slurm 集群 10 节点手操部署文档

**机器清单**

| 节点 | 角色 | IP | CPU | 内存 | GPU / Gres | 分区 |
| --- | --- | --- | --- | --- | --- | --- |
| master | slurmctld + slurmdbd + MariaDB + NTP + CPU 计算 | 192.168.19.95 | 64 核(AMD 9124) | 251G | — | cpu（默认） |
| storage | NFS 服务端 `/mnt/sdb`(220T RAID5) + CPU 计算 | 192.168.19.65 | 64 核 | 251G | — | cpu |
| gpu1 | slurmd（GPU + 高核CPU） | 192.168.19.82 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |
| gpu2 | slurmd | 192.168.18.184 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |
| gpu3 | slurmd | 192.168.19.205 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |
| gpu4 | slurmd | 192.168.19.166 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |
| gpu5 | slurmd | 192.168.19.121 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |
| gpu6 | slurmd | 192.168.18.82 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |
| gpu7 | slurmd | 192.168.19.248 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |
| gpu8 | slurmd | 192.168.18.235 | 384 核 | 1TB | 8×A100 / `gpu:8` | gpu, cpugpu |

> 记账：ClusterName=`aiclust`；Slurm 用户 `slurm`(uid/gid 64030)；记账库 `slurm_acct_db`，库账户 `slurm`/`Slurm@123456`；作业记账账号 `ps → aiclust`。
> 分区：`cpu`(master+storage,默认) / `gpu`(gpu1-8) / `cpugpu`(gpu1-8 的高核纯 CPU)。

---

## 1. 部署前只读检查（每台）

逐台登录核对，不改动系统：

```bash
hostname && grep PRETTY_NAME /etc/os-release
lscpu | grep -E '^(CPU\(s\)|Model name|Socket|Core|Thread|NUMA node\(s\))'; nproc
grep -E '^(MemTotal|SwapTotal)' /proc/meminfo
nvidia-smi --query-gpu=count,name,memory.total --format=csv,noheader || echo NO_NVIDIA
lsblk -o NAME,SIZE,TYPE,MOUNTPOINT,MODEL; df -hT | grep -Ev 'tmpfs|loop'
date; timedatectl | grep -E 'synchronized|Time zone'
command -v slurmd sbatch scontrol sinfo sacct munge unmunge || echo '无旧版残存'
```

> 核对：storage 的 `/dev/sdb`(12×22TB RAID5→220T) 与挂载点；gpu 节点的 `nvidia-smi` 显示 8 张 A100；时间基本一致；无旧版 slurm/munge。

---

## 2. 基础环境（每台）

### 2.1 主机名
```bash
# [每台] 换成各自名称：master/storage/gpu1~gpu8
sudo hostnamectl set-hostname gpu1 && hostname
```

### 2.2 统一 hosts（每台追加全集）
```bash
# [每台] 追加到 /etc/hosts
cat <<'EOF' | sudo tee -a /etc/hosts
192.168.19.95 master
192.168.19.65 storage
192.168.19.82 gpu1
192.168.18.184 gpu2
192.168.19.205 gpu3
192.168.19.166 gpu4
192.168.19.121 gpu5
192.168.18.82 gpu6
192.168.19.248 gpu7
192.168.18.235 gpu8
EOF
```

### 2.3 关防火墙 + 提 limits（每台）
```bash
sudo systemctl stop ufw; sudo systemctl disable ufw; sudo iptables -F
printf '* soft nofile 65535\n* hard nofile 65535\n* soft nproc 65535\n* hard nproc 65535\n' | sudo tee -a /etc/security/limits.conf
```

---

## 3. 时间同步 chrony（每台）

```bash
sudo apt-get update -y && sudo apt-get install -y chrony
```

**master**（NTP 源）：
```bash
# [master]
cat <<'EOF' | sudo tee /etc/chrony/chrony.conf
pool ntp.ubuntu.com iburst
allow 192.168.19.0/24
allow 192.168.18.0/24
local stratum 10
EOF
```
**storage / gpu1-8**（客户端指向 master）：
```bash
# [storage/gpu1-8]
cat <<'EOF' | sudo tee /etc/chrony/chrony.conf
server master iburst
EOF
```
每台核对：
```bash
sudo systemctl restart chrony; sleep 3
chronyc sources | head      # 客户端应看到 ^* master
timedatectl | grep -E 'synchronized|Time zone'
sudo timedatectl set-local-rtc 0        # RTC 统一 UTC
timedatectl | grep RTC                  # -> RTC in local TZ: no
```

---

## 4. NFS 共享（storage 导出 → 其余 9 台挂 /shared）

### 4.1 storage 服务端
```bash
# [storage]
sudo apt-get install -y nfs-kernel-server
sudo mkdir -p /mnt/sdb/home /mnt/sdb/slurm-logs /mnt/sdb/scratch
sudo chown ps:ps /mnt/sdb/slurm-logs /mnt/sdb/scratch && sudo chmod 770 /mnt/sdb/slurm-logs /mnt/sdb/scratch
grep -q '^/mnt/sdb ' /etc/exports 2>/dev/null || echo '/mnt/sdb *(rw,sync,no_root_squash,no_subtree_check)' | sudo tee -a /etc/exports
sudo exportfs -ra && sudo systemctl enable --now nfs-server && sudo exportfs -v
```
> `slurm-logs`=作业输出统一落点、`scratch`=普通用户临时共享区（均 ps:ps 770）。

### 4.2 客户端（master / gpu1-8）
```bash
# [master/gpu1-8]
sudo apt-get install -y nfs-common
sudo mkdir -p /shared
echo 'storage:/mnt/sdb /shared nfs defaults,nofail,_netdev,bg,hard,intr,x-systemd.mount-timeout=30 0 0' | sudo tee -a /etc/fstab
sudo mount -a
df -hT | grep /shared          # 应见 storage:/mnt/sdb 220T
```
> 必须 `nofail,_netdev,bg`，否则开机 NFS 未就绪会掉 emergency（见 §13.1）。

---

## 5. MUNGE 认证（每台，共享同一把 key）

### 5.1 每台安装 + 建目录
```bash
sudo apt-get install -y munge libmunge-dev libmunge2
sudo mkdir -p /var/lib/munge /run/munge /var/log/munge /etc/munge
sudo chown -R munge:munge /var/lib/munge /run/munge /var/log/munge /etc/munge
```

### 5.2 master 生成密钥
```bash
# [master]
if [ ! -s /etc/munge/munge.key ]; then sudo dd if=/dev/urandom of=/etc/munge/munge.key bs=1024 count=1 status=none; fi
sudo chown munge:munge /etc/munge/munge.key; sudo chmod 400 /etc/munge/munge.key
sudo base64 -w0 /etc/munge/munge.key     # 记下 KEYB64，下面用
```

### 5.3 其余 9 台写入同一密钥
```bash
# [storage/gpu1-8] KEYB64 替换为上面记的串
echo 'KEYB64' | base64 -d | sudo tee /etc/munge/munge.key >/dev/null
sudo chown munge:munge /etc/munge/munge.key; sudo chmod 400 /etc/munge/munge.key
```

### 5.4 每台启动 + 校验
```bash
sudo systemctl enable munge && sudo systemctl restart munge   # 服务名 munge.service
sleep 2; systemctl is-active munge
sha256sum /etc/munge/munge.key | awk '{print $1}'            # 10 台应完全一致
echo 'test' | munge -n | unmunge 2>&1 | grep STATUS          # -> Success
# 跨节点
echo 'test' | munge -n | ssh gpu1 'unmunge 2>&1 | grep STATUS'
```

---

## 6. 编译 Slurm 24.11.6（master）+ scp 分发

### 6.1 依赖（master）
```bash
sudo apt-get install -y build-essential gcc make hwloc libhwloc-dev libmunge-dev libssl-dev \
  libcurl4-openssl-dev libjson-c-dev liblua5.3-dev \
  libdbus-1-dev libbpf-dev libelf-dev linux-libc-dev pkg-config libmariadb-dev libmariadb-dev-compat
```

### 6.2 下载 + 编译（master）
```bash
mkdir -p /opt/slurm-build && cd /opt/slurm-build
if [ ! -s slurm-24.11.6.tar.bz2 ] || [ "$(stat -c%s slurm-24.11.6.tar.bz2)" -lt 1000000 ]; then
  curl -fL --retry 3 -o slurm-24.11.6.tar.bz2 https://download.schedmd.com/slurm/slurm-24.11.6.tar.bz2
fi
tar xjf slurm-24.11.6.tar.bz2 && cd slurm-24.11.6
make distclean >/dev/null 2>&1 || true
./configure --prefix=/usr/local --sysconfdir=/etc/slurm --datarootdir=/usr/local/share \
  --with-munge --with-hwloc --enable-cgroupv2=yes --with-mysql_config=/usr/bin
make -j$(nproc) && sudo make install
ls -la /usr/local/lib/slurm/cgroup_v2.so /usr/local/lib/slurm/accounting_storage_mysql.so
```
> `--with-mysql_config=/usr/bin` 必须是**目录**；`--enable-cgroupv2=yes` 出 cgroup_v2 插件。两插件缺失会致 slurmd/slurmdbd 启动失败。

### 6.3 打包 + scp 分发
```bash
# [master]
sudo tar czf /tmp/slurm_usr_local.tgz -C / /usr/local
ls -lh /tmp/slurm_usr_local.tgz          # 约 129M，此包存 /shared 备用：cp 到 /shared/slurm-install-swing.tar.gz
# 从任何能 ssh 到各机的地方：把包 scp 到 storage/gpu1-8
scp /tmp/slurm_usr_local.tgz ps@storage:/tmp/
scp /tmp/slurm_usr_local.tgz ps@gpu1:/tmp/
# ... 同理 gpu2~gpu8
```
各目标节点解包：
```bash
# [storage/gpu1-8]
cd / && sudo tar xzf /tmp/slurm_usr_local.tgz -C /
echo '/usr/local/lib' | sudo tee /etc/ld.so.conf.d/slurm.conf
sudo ldconfig
/usr/local/sbin/slurmd -V                # -> slurm 24.11.6
ldd /usr/local/sbin/slurmd | grep -i hwloc   # 确认 libhwloc.so.15 已解析
```
> 若报 `libhwloc.so.15` 缺失：`sudo apt-get install -y libhwloc15 libnuma1 libjson-c5`。

---

## 7. 用户 + systemd 单元（每台）

```bash
getent group slurm >/dev/null || sudo groupadd -g 64030 slurm
id slurm >/dev/null 2>&1 || sudo useradd -r -g slurm -u 64030 -s /bin/false slurm
sudo mkdir -p /var/spool/slurm/ctld /var/spool/slurm/d /var/log/slurm /var/run/slurm /etc/slurm
sudo chown -R slurm:slurm /var/spool/slurm /var/log/slurm /var/run/slurm
sudo chmod 755 /var/spool/slurm /var/run/slurm
id slurm
```

**slurmctld.service**（仅 master）：
```bash
# [master]
cat <<'EOF' | sudo tee /etc/systemd/system/slurmctld.service
[Unit]
Description=Slurm controller daemon
After=network.target munge.service
Wants=munge.service
[Service]
Type=forking
ExecStart=/usr/local/sbin/slurmctld
ExecReload=/bin/kill -HUP $MAINPID
EnvironmentFile=-/etc/default/slurmctld
PIDFile=/run/slurmctld.pid
LimitNOFILE=3141592
[Install]
WantedBy=multi-user.target
EOF
```
**slurmd.service**（每台）：
```bash
# [每台]
cat <<'EOF' | sudo tee /etc/systemd/system/slurmd.service
[Unit]
Description=Slurm node daemon
After=network.target munge.service local-fs.target
Wants=munge.service
[Service]
Type=simple
ExecStart=/usr/local/sbin/slurmd -D $SLURMD_OPTIONS
ExecReload=/bin/kill -HUP $MAINPID
EnvironmentFile=-/etc/default/slurmd
KillMode=process
LimitNOFILE=131072
[Install]
WantedBy=multi-user.target
EOF
```

---

## 8. 配置文件（本集群最终值）

> 写好后把 `slurm.conf`/`cgroup.conf` 用 scp 同步到每台保持全集群一致（`md5sum` 对齐）；`gres.conf` 仅 gpu1-8。

### 8.1 `/etc/slurm/slurm.conf`（每台，内容一致）
```bash
# [每台]
cat <<'EOF' | sudo tee /etc/slurm/slurm.conf
# Slurm 24.11.6 aiclust 10 节点（cgroup/v2 + slurmdbd 记账）
ClusterName=aiclust
SlurmctldHost=master
SlurmUser=slurm
SlurmdUser=root
AuthType=auth/munge
CredType=cred/munge
GresTypes=gpu

MpiDefault=none
ProctrackType=proctrack/cgroup
TaskPlugin=task/affinity,task/cgroup
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
JobAcctGatherType=jobacct_gather/cgroup
JobAcctGatherFrequency=task=30

AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=master
AccountingStoragePort=6819

NodeName=master   NodeAddr=192.168.19.95   CPUs=64  RealMemory=250000 State=UNKNOWN
NodeName=storage  NodeAddr=192.168.19.65   CPUs=64  RealMemory=250000 State=UNKNOWN
NodeName=gpu1     NodeAddr=192.168.19.82   CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN
NodeName=gpu2     NodeAddr=192.168.18.184  CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN
NodeName=gpu3     NodeAddr=192.168.19.205  CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN
NodeName=gpu4     NodeAddr=192.168.19.166  CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN
NodeName=gpu5     NodeAddr=192.168.19.121  CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN
NodeName=gpu6     NodeAddr=192.168.18.82   CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN
NodeName=gpu7     NodeAddr=192.168.19.248  CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN
NodeName=gpu8     NodeAddr=192.168.18.235  CPUs=384 RealMemory=1000000 Gres=gpu:8 State=UNKNOWN

# 分区（显式列举节点，勿用区间写法）
PartitionName=cpu   Nodes=master,storage Default=YES MaxTime=INFINITE State=UP AllowGroups=ALL
PartitionName=gpu   Nodes=gpu1,gpu2,gpu3,gpu4,gpu5,gpu6,gpu7,gpu8 MaxTime=INFINITE State=UP AllowGroups=ALL
PartitionName=cpugpu Nodes=gpu1,gpu2,gpu3,gpu4,gpu5,gpu6,gpu7,gpu8 MaxTime=INFINITE State=UP AllowGroups=ALL
EOF
```
> gpu 节点资源识别为 `Sockets=384 CoresPerSocket=1 ThreadsPerCore=1`（真实 2×EPYC 超线程）——CPU 总量/内存/Gres 正确、调度正常，属已知无害项，超线程对调度不可见。

### 8.2 `/etc/slurm/gres.conf`（仅 gpu1-8）
```bash
# [gpu1-8]
echo 'Name=gpu Type=A100 File=/dev/nvidia[0-7]' | sudo tee /etc/slurm/gres.conf
```

### 8.3 `/etc/slurm/cgroup.conf`（每台）
```bash
# [每台]
cat <<'EOF' | sudo tee /etc/slurm/cgroup.conf
CgroupPlugin=cgroup/v2
CgroupMountpoint=/sys/fs/cgroup
ConstrainCores=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=no
ConstrainDevices=no
AllowedRAMSpace=98
EOF
```

### 8.4 校验
```bash
# [master]
sudo /usr/local/sbin/slurmctld -t      # 无输出=语法正确
```

---

## 9. 启动顺序（master 先 slurmctld，后各台 slurmd）

```bash
# [master]
sudo systemctl daemon-reload && sudo systemctl enable --now slurmctld
sleep 2; systemctl is-active slurmctld
# [每台]
sudo systemctl daemon-reload && sudo systemctl enable --now slurmd
sleep 5
# [master]
sinfo; scontrol ping
```
> 若 slurmd 先于 slurm.conf 就绪而退出，§13.3 的 `Restart=on-failure` 会随后自动拉起。

---

## 10. 记账：MariaDB + slurmdbd（master）

### 10.1 装 MariaDB + InnoDB 调优
```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y mariadb-server mariadb-client
sudo systemctl enable --now mariadb; sleep 2; systemctl is-active mariadb
cat <<'EOF' | sudo tee /etc/mysql/mariadb.conf.d/99-slurm.cnf
[mysqld]
innodb_buffer_pool_size=8G
innodb_log_file_size=512M
innodb_lock_wait_timeout=900
EOF
sudo systemctl restart mariadb
```

### 10.2 建库 / 用户 / 授权
```bash
sudo mysql -e "CREATE DATABASE IF NOT EXISTS slurm_acct_db; CREATE USER IF NOT EXISTS 'slurm'@'localhost' IDENTIFIED BY 'Slurm@123456'; GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost'; FLUSH PRIVILEGES;"
```

### 10.3 slurmdbd.conf（master）
```bash
# [master]
cat <<'EOF' | sudo tee /etc/slurm/slurmdbd.conf
AuthType=auth/munge
DbdHost=master
StorageType=accounting_storage/mysql
StorageHost=localhost
StorageUser=slurm
StoragePass=Slurm@123456
StorageLoc=slurm_acct_db
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurm/slurmdbd.pid
SlurmUser=slurm
EOF
sudo chown slurm:slurm /etc/slurm/slurmdbd.conf && sudo chmod 600 /etc/slurm/slurmdbd.conf
```

### 10.4 slurmdbd.service（master）
```bash
# [master]
cat <<'EOF' | sudo tee /etc/systemd/system/slurmdbd.service
[Unit]
Description=Slurm DBD accounting daemon
After=network.target munge.service mariadb.service
Wants=munge.service
[Service]
Type=forking
User=slurm
Group=slurm
ExecStart=/usr/local/sbin/slurmdbd
ExecReload=/bin/kill -HUP $MAINPID
EnvironmentFile=-/etc/default/slurmdbd
PIDFile=/var/run/slurm/slurmdbd.pid
LimitNOFILE=131072
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now slurmdbd
sleep 3; systemctl is-active slurmdbd; sudo tail -5 /var/log/slurm/slurmdbd.log
```

### 10.5 启用记账 + CLUSTER ID 修复
```bash
# [master]
ls -la /usr/local/lib/slurm/accounting_storage_mysql.so
sudo systemctl restart slurmdbd; sleep 3
sudo systemctl restart slurmctld; sleep 3
# 若 slurmctld 报 CLUSTER ID MISMATCH：
sudo systemctl stop slurmctld; sleep 1
sudo rm -f /var/spool/slurm/ctld/clustername
sudo systemctl start slurmctld; sleep 3
sudo tail -8 /var/log/slurm/slurmctld.log   # creating clustername file: aiclust ClusterID=...
```

---

## 11. 正式记账账号（须用 root/sudo）

> 普通用户非 SlurmDBD admin，`sacctmgr -i` 直接跑会报 `Only admins/operators/coordinators can add accounts`，**必须 `sudo` 以 root 执行**。

```bash
# [master]
sudo sacctmgr -i add account name=aiclust cluster=aiclust
sudo sacctmgr -i add user   name=ps       account=aiclust
sudo sacctmgr -i modify user name=ps        set DefaultAccount=aiclust
sacctmgr -n show assoc format=Cluster,Account,User,DefaultAccount,QOS    # 应含 aiclust/aiclust/ps
sacctmgr -n show user name=ps format=User,Account,DefaultAccount,AdminLevel   # 默认 aiclust
```

---

## 12. 自愈与加固（每台按角色）

### 12.1 NFS `nofail,_netdev,bg`（已在 §4.2 写入）
核对：`grep /shared /etc/fstab`。勿用 `x-systemd.automount`；先重启 storage 再客户端，`bg` 自动重试。

### 12.2 `/run/slurm` tmpfiles（每台按角色）
```bash
# [master] 控制端版（建 ctld）
sudo mkdir -p /etc/tmpfiles.d
cat <<'EOF' | sudo tee /etc/tmpfiles.d/slurm.conf
d /run/slurm 0755 slurm slurm -
d /run/slurm/ctld 0755 slurm slurm -
EOF
sudo systemd-tmpfiles --create /etc/tmpfiles.d/slurm.conf
```
```bash
# [storage/gpu1-8] 计算节点版（建 d）
sudo mkdir -p /etc/tmpfiles.d
cat <<'EOF' | sudo tee /etc/tmpfiles.d/slurm.conf
d /run/slurm 0755 slurm slurm -
d /run/slurm/d 0755 slurm slurm -
EOF
sudo systemd-tmpfiles --create /etc/tmpfiles.d/slurm.conf
```
> 两版差异有意为之。⚠️ 慎用 `printf '%s' '...\n...'` 或 python `repr` 传值——会写入字面量 `\nd`，tmpfiles 报 `Invalid age '-nd'`；务必用 here-doc 真实换行。

### 12.3 服务自愈 `Restart=on-failure`（每台对应服务）
```bash
# [每台] slurmd drop-in
sudo mkdir -p /etc/systemd/system/slurmd.service.d
cat <<'EOF' | sudo tee /etc/systemd/system/slurmd.service.d/override.conf
[Service]
Restart=on-failure
RestartSec=10
EOF
sudo systemctl daemon-reload && sudo systemctl restart slurmd
# [master] 再为 slurmctld / slurmdbd 各加一份同名 override（把 slurmd 换成 slurmctld/slurmdbd）
```
> 新 GPU 节点首次 slurmd 报 `cpu cgroup controller is not available` 属启动竞态，`Restart=on-failure` 会自动重试自愈，无需人工。

### 12.4 Swap 关闭（每台）
```bash
sudo swapoff -a
sudo cp /etc/fstab /etc/fstab.bak.$(date +%Y%m%d%H%M%S)
sudo sed -i '\#/swapfile#d' /etc/fstab
sudo systemctl mask '*.swap'
swapon --show       # 应无输出；free -h 的 Swap=0B
```

### 12.5 master → 各节点免密 SSH（master 执行，运维便利）
```bash
# [master] 生成本机密钥（如无）
if [ ! -f ~/.ssh/id_ed25519 ]; then ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519 -q; fi
# ps 家目录各节点本地、非 NFS，公钥须逐台写；在每台 authorized_keys 追加 master 的 id_ed25519.pub 内容
# 或：对每台执行 ssh-copy-id（首次仍输一次口令）
for h in storage gpu1 gpu2 gpu3 gpu4 gpu5 gpu6 gpu7 gpu8; do ssh $h 'true'; done
# 验证
ssh gpu8 hostname
```

---

## 13. 端到端验收（master）

```bash
sinfo                        # cpu×2 + gpu×8 + cpugpu×8 全 idle
scontrol ping                # UP
srun -p cpu -N2 -n2 hostname
srun -p gpu --gres=gpu:1 nvidia-smi --query-gpu=index,name,memory.total --format=csv,noheader
srun -p cpugpu -n 300 hostname
squeue
srun -p cpu --account=aiclust -n2 hostname; sleep 2
sacct -S now-1hour --format=JobID,State,Elapsed,NodeList,AllocTRES%20
sudo mysql -e "select count(*) from slurm_acct_db.aiclust_job_table;"
```
> 全节点 `idle`、作业 `COMPLETED`、`sacct` 含 billing/cpu/mem、记账表有条目即通过。

---

## 14. 关键坑速查（本集群实测）

| 坑 | 现象 | 解决 |
| --- | --- | --- |
| 源码下载不完整 | 下载后过小 | `curl -fL --retry 3` + `stat -c%s < 1000000` 校验 |
| 缺 cgroup_v2 插件 | `proctrack/cgroup` 启动失败 | `--enable-cgroupv2=yes` 重编 |
| 缺 MySQL 插件 | slurmdbd 无法加载 mysql | `--with-mysql_config=/usr/bin`(目录) 重编 |
| `--with-mysql_config` 传单文件 | 仍不带 mysql 插件 | 传**目录** `/usr/bin` |
| SlurmdUser 非 root | cgroup 约束不生效 | `SlurmdUser=root` |
| 分区节点区间写法 | `Nodes=gpu1-8` 解析异常 | 显式 `gpu1,gpu2,...,gpu8` |
| MUNGE 服务名拼错 | 依赖等待错误 | 服务名 `munge.service` |
| slurmdbd 90s 超时被杀 | 写不出 pidfile | §12.2 tmpfiles 重建 `/run/slurm` |
| 大文件分发 SSH 重置 | base64 分发卡住 | 用 **scp/SFTP 直传** |
| CLUSTER ID MISMATCH | 启用记账后 slurmctld 起不来 | `rm -f /var/spool/slurm/ctld/clustername` 后重启（DB 为准） |
| `libhwloc.so.15` 缺失 | slurmd 打不开 | `apt-get install libhwloc15 libnuma1` |
| 新节点首次 slurmd 报 cgroup | `cpu cgroup controller is not available` | 不需处理，`Restart=on-failure` 自愈 |
| NFS 缺 nofail | 开机掉 emergency | `nofail,_netdev,bg` |
| `sacctmgr` 报 Only admins | 非 admin | `sudo sacctmgr`（root）执行 |
| tmpfiles 报 `Invalid age '-nd'` | 配置文件写成单行 | here-doc 真实换行重写 |
| 普通用户写 `/shared` 失败 | 根为 root:755 | 用 `/shared/scratch`、`/shared/slurm-logs`(ps:ps 770) |

---

## 15. 逐台填写核对表（交付/验收）

| 机名 | IP | 角色 | 基础 | chrony | NFS | MUNGE | slurmd | slurm.conf | tmpfiles | 自愈 | swap | RTC | ✔/✘ |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| master | 192.168.19.95 | slurmctld/slurmdbd/MariaDB/cpu | | | 客户端 | | | | 控制端版 | | | | ☐ |
| storage | 192.168.19.65 | NFS 服务端/cpu | | | 服务端 | | | | 计算版 | | | | ☐ |
| gpu1 | 192.168.19.82 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |
| gpu2 | 192.168.18.184 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |
| gpu3 | 192.168.19.205 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |
| gpu4 | 192.168.19.166 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |
| gpu5 | 192.168.19.121 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |
| gpu6 | 192.168.18.82 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |
| gpu7 | 192.168.19.248 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |
| gpu8 | 192.168.18.235 | slurmd | | | 客户端 | | | | 计算版 | | | | ☐ |

**账记确认**：`sacctmgr show assoc` 含 `aiclust aiclust ps` 且默认账户=`aiclust`；`sacct --brief` 有 COMPLETED。
**待现场核验**：storage 的 220T RAID5(AVAGO MR9361-8i) 需装 storcli 或 IPMI 确认阵列健康。

---

*真实值手操版，随 aiclust 部署/验收使用；不含任何自定义脚本。逐台命令均可直接复制执行。*