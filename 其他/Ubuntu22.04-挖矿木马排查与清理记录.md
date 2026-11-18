# Ubuntu 22.04 CPU 长期约 50% 占用：排查与清理记录

> 机器环境：AMD EPYC 7Y43（48 核 / 96 线程）、约 256GB 内存、Ubuntu 22.04  
> 现象：空闲时整体 CPU 约 50%，重启后依旧；`top`/`ps` 看不到明显高占用进程  
> 结论：隐藏式 CPU 挖矿木马 + systemd 持久化；清理服务与二进制后恢复正常

---

## 1. 现象

| 项目 | 表现 |
|------|------|
| CPU | `top` 中 `us≈50%`，`id≈50%`，`wa≈0` |
| Load | 长期约 `48.x`（96 线程机器上约等于半机满载） |
| 内存 | 几乎空闲，非内存压力 |
| 进程列表 | 按 CPU 排序也看不到能解释 50% 的进程 |
| 重启 | 重启后依旧约 50% |

易误判为：系统空转、驱动问题、或“正常半满算力”。

---

## 2. 排查过程（关键步骤）

### 2.1 确认核数与负载是否匹配

```bash
nproc          # 96
cat /proc/loadavg
```

`load≈48` + `idle≈50%` 与「约一半逻辑核在忙」一致，但当时**看不到对应业务进程**。

### 2.2 用户态进程加总对不上

```bash
ps -eo user,pcpu | awk '{u[$1]+=$2} END{for(i in u) printf "%6.1f  %s\n", u[i], i}' | sort -rn | head
```

结果：各用户 `%CPU` 加总远小于 50%（例如仅约十几个百分点）。

`R` 状态进程几乎没有；无大量 `D` 状态（非磁盘卡住）。

### 2.3 用 `/proc/stat` 证明“真的在忙”

未安装 `mpstat` 时用 Python 采样：

- 约 **48 个逻辑核 ≥80%（多为 100%）**
- 约一半核空闲
- 平均利用率约 **50%**
- 忙核几乎全是 **`us=100%`（用户态）**，不是内核/`kworker` 为主

结论：CPU 会计真实，不是 `top` 误报。

### 2.4 用 sysrq 抓各 CPU 调用栈（实锤）

```bash
sudo sh -c 'echo l > /proc/sysrq-trigger'
sudo dmesg -T | tail -400
```

NMI backtrace 显示大量逻辑核在跑同一进程：

- 进程名：`2cb04c2d`（随机十六进制名）
- `RIP: 0033:...`（用户态）
- 多 PID 同时占用

该进程对常规 `ps`/`top` **不可见或极难对应**，属于带隐藏能力的挖矿样本。

### 2.5 临时止血

杀掉相关挖矿进程后：

- `id` 回到约 `99%`
- 1 分钟 load 迅速下降  

说明占用由该用户态负载造成；**但仅杀进程不够**（有自启）。

---

## 3. 根因

### 3.1 挖矿进程

- 名称：`2cb04c2d`
- 类型：CPU 挖矿（约吃满 48 个逻辑核，整体显示约 50%）
- 特征：隐藏进程、随机名、高用户态占用

本次样本表现为 **CPU 挖矿**；未发现其占用 GPU 的证据。GPU 是否另有木马需用 `nvidia-smi` 另行确认，与本次 CPU 问题无直接证据关联。

### 3.2 持久化（真正“清干净”的关键）

发现并清除的 systemd 服务：

```ini
# /lib/systemd/system/tmuxinfo9f7fc4e9.service
[Service]
Type=simple
User=root
ExecStart=/usr/bin/tmuxinfo9f7fc4e9d90e5aa5 tmuxinfo9f7fc4e9
Restart=always
RestartSec=4h
```

| 项 | 内容 |
|----|------|
| 伪装 | 名称像 `tmuxinfo…`，实为木马 |
| 二进制 | `/usr/bin/tmuxinfo9f7fc4e9d90e5aa5`（静态链接 ELF，stripped） |
| 策略 | root 运行；失败/退出后约每 4 小时拉起；开机启用 |
| 旁证 | 该服务曾累计 CPU 约 **3 天+** |

因此：只 `kill` 进程 → CPU 暂降；重启或等待后仍可能回来。

### 3.3 其它篡改（同一感染面）

| 发现 | 说明 |
|------|------|
| 假 `/usr/bin/chattr` | 仅 76 字节脚本，打印假帮助（含 `vmlinux1`），废掉真正 `chattr` |
| 文件保护 | 假 `chattr` 曾带 **`+a`（只追加）**，导致无法 `mv`/`rm`（报 Operation not permitted） |
| apt / systemd | 曾出现 systemd 版本不一致、`systemctl` 升级失败等，安装软件受阻 |
| `reboot` 异常 | 曾出现执行 `reboot` 却打出 `systemctl` 单元列表（未真正重启）的情况 |

这些说明系统基础命令/包管理也曾被影响，**不能仅视为“多了一个挖矿进程”**。

---

## 4. 清理步骤（已执行要点）

### 4.1 停止并删除木马服务与程序

```bash
sudo systemctl stop tmuxinfo9f7fc4e9.service
sudo systemctl disable tmuxinfo9f7fc4e9.service
sudo rm -f /lib/systemd/system/tmuxinfo9f7fc4e9.service
sudo rm -f /etc/systemd/system/tmuxinfo9f7fc4e9.service
sudo rm -f /usr/bin/tmuxinfo9f7fc4e9d90e5aa5
sudo systemctl daemon-reload
sudo systemctl reset-failed
```

确认：

```bash
systemctl status tmuxinfo9f7fc4e9.service   # 应为 could not be found
sudo find /usr/bin /lib/systemd/system /etc/systemd/system -iname '*tmuxinfo*'
```

### 4.2 恢复被篡改的 `chattr`

假文件属性曾为 `-----a--------e-------`（关键是 **`a`**，不是只有 `i`）。

```bash
cd /tmp
apt-get download e2fsprogs
dpkg-deb -x e2fsprogs_*.deb /tmp/e2fsprogs-extract

# 用解压出的真 chattr 去掉保护位
sudo /tmp/e2fsprogs-extract/usr/bin/chattr -iauc /usr/bin/chattr
sudo rm -f /usr/bin/chattr
sudo cp -a /tmp/e2fsprogs-extract/usr/bin/chattr /usr/bin/chattr
sudo cp -a /tmp/e2fsprogs-extract/usr/bin/lsattr /usr/bin/lsattr
sudo chmod 755 /usr/bin/chattr /usr/bin/lsattr
file /usr/bin/chattr   # 应为 ELF，而非 ASCII text
```

### 4.3 其它排查（当时未发现明显残留）

- `/tmp`、`/var/tmp`、`/dev/shm`：无挖矿可执行文件
- 系统 crontab / root crontab：空或无命中可疑下载命令
- Docker：无容器/无镜像
- `ld.so.preload`：空文件
- 按名查找 `2cb04c2d` / `xmrig` / `kdevtmpfsi` 等：未找到落地文件（可能已删除、改名或仅由上述 systemd 拉起）

---

## 5. 验证

清理并杀进程后：

- `busy_cores>=80%` → `0`
- `top`：`id≈99%`
- load 1 分钟值回落；5/15 分钟需一段时间衰减

建议持续观察：

```bash
cat /proc/loadavg
# 正常空闲时 1 分钟 load 应远低于 ~48
```

---

## 6. 经验总结

1. **整体 50% + load≈核数一半**：先算 `nproc`，再看是真负载还是异常。  
2. **`ps` 加总 << `top` 的 us%**：高度怀疑隐藏进程/短命进程/会计异常，优先用 `/proc/stat` 与 `perf`/`sysrq-l`。  
3. **杀进程只是止血**；必须查 `systemd` / cron / SSH 密钥 / 被替换的系统命令。  
4. 伪装成 `tmuxinfo*` 的 unit + `/usr/bin` 下随机后缀二进制，是典型持久化手法。  
5. `Operation not permitted` 删不掉文件时，用**完好的** `lsattr`/`chattr` 看是否有 `i`/`a` 等属性。  
6. 出现进程隐藏 + 系统命令篡改时，**手工清理无法保证 100% 干净**；重要机器建议备份后重装，并轮换密码与 SSH 密钥。

---

## 7. 建议的后续动作（安全加固）

- [ ] 修改 `vip` / `root` / `zqz` 等账号密码  
- [ ] 检查各用户 `~/.ssh/authorized_keys`，清除陌生公钥  
- [ ] 修复 apt/systemd 依赖（若仍无法正常 `apt install`）  
- [ ] 确认 `reboot`/`shutdown`/`systemctl` 等关键命令是否为官方文件  
- [ ] 观察数日 load/CPU；必要时重装系统  
- [ ] 审查入口：弱口令 SSH、暴露端口、共用账号、来历不明脚本等  

---

## 8. 时间线摘要

1. 发现空闲约 50% CPU，重启无效  
2. 确认 96 线程；load≈48；进程列表无法解释  
3. `/proc/stat` 显示约 48 核用户态打满  
4. `sysrq-l` + `dmesg` 定位隐藏进程 `2cb04c2d`  
5. 杀进程后 CPU 恢复；查找持久化  
6. 定位并删除 `tmuxinfo9f7fc4e9.service` 与对应二进制  
7. 发现并恢复被 `+a` 保护的假 `chattr`  
8. 重启验证连通性/服务（若 SSH 异常，需控制台检查 `ssh` 服务是否拉起）  

---

*文档用途：问题回顾与同类故障排查参考。涉及具体主机名、账号、内网地址时请按需脱敏。*
