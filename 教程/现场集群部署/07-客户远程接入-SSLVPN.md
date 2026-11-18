# 07 · 客户远程接入（SSL VPN）

> **场景**：客户在**深圳**、机器在**杭州**。客户需要远程**管理 + 使用**这些机器 ——
> 既开 BMC 带外页面（开关机、看硬件、KVM），也 SSH 登录业务网节点跑任务、传数据。
> ⇒ 访问范围是**两个网段**：管理网 `192.168.10.0/24` + 业务网 `192.168.11.0/24`。
> **配套**：《配置命令执行手册.md》阶段 8；《HPC集群网络配置方案.md》§5.6；客户可直接阅读的版本见 `docs/客户使用说明.md`。

---

## 1. 方案选型

| 方案 | 结论 |
|---|---|
| **SSL VPN「IP 接入」（网络扩展）** | ✅ **采用**。需要 SSH / 传数据这类非 Web 的原始 IP 流量，客户端拿到虚拟 IP 后像在局域网内一样工作 |
| 把 BMC 端口逐个映射到公网 | ❌ 不采用 —— 暴露面更大，且无法覆盖业务网 |
| IPSec VPN | 未采用（客户侧部署成本更高，且当前需求用 SSL VPN 已满足） |

**入口**：`https://60.191.84.165:4430`（端口 4430 是为了**避开 NAT Server 的 443**）。
**客户端**：Windows 装 **iNode**（从 SSL VPN 网关登录页下载的那份，是配套版本）。

---

## 2. 服务端配置（阶段 8 ①–⑥）

```
system-view

# ① 建客户端地址池（⚠ 地址池在【系统视图】建；context 里的 ip-tunnel address-pool 是"引用"）
sslvpn ip address-pool vpnpool 192.168.99.1 192.168.99.100

# ② 隧道接口（客户端侧网关地址）
interface SSLVPN-AC 1
 ip address 192.168.99.254 255.255.255.0
quit

# ③ SSL 网关：挂【在用】的电信接口地址 .165，端口 4430
sslvpn gateway remotegw
 ip address 60.191.84.165 port 4430
 service enable
quit

# ④ ACL：限定 VPN 用户可访问的目的网段（IP 接入的【第一道过滤：按目的 IP】）
acl advanced 3999
 rule 0 permit ip destination 192.168.10.0 0.0.0.255
 rule 5 permit ip destination 192.168.11.0 0.0.0.255
 rule 10 permit ip destination 192.168.99.0 0.0.0.255    # 可选：让运维经 VPN 访问设备自身（隧道网关）
quit

# ⑤ 访问实例：地址池引用 + 下发路由 + 策略组
sslvpn context remote
 gateway remotegw
 ip-tunnel interface SSLVPN-AC 1
 ip-route-list intra
  include 192.168.10.0 24
  include 192.168.11.0 24
 quit
 ip-tunnel address-pool vpnpool mask 24
 ip-tunnel dns-server primary 223.5.5.5
 policy-group customer
  ip-tunnel access-route ip-route-list intra     # 下发给客户端的路由
  filter ip-tunnel acl 3999                       # ⚠ 必配：不加则不做访问控制
 quit
 service enable
quit

# ⑥ 客户账号（每人一号）
local-user cust01 class network
 password simple <强口令>
 service-type sslvpn
 authorization-attribute sslvpn-policy-group customer
 authorization-attribute user-role network-operator
quit
```

| 配置对象 | 名称 | 关键值 |
|---|---|---|
| 地址池 | `vpnpool` | `192.168.99.1` – `192.168.99.100` |
| 隧道接口 | `interface SSLVPN-AC 1` | `192.168.99.254/24` |
| SSL 网关 | `remotegw` | `60.191.84.165 : 4430`，`service enable` |
| 访问控制列表 | `acl advanced 3999` | 目的 IP 白名单（管理网 / 业务网 / 隧道网段） |
| 路由列表 | `ip-route-list intra` | `include 192.168.10.0 24`、`include 192.168.11.0 24` |
| 策略组 | `policy-group customer` | `ip-tunnel access-route ip-route-list intra` + **`filter ip-tunnel acl 3999`** |
| 客户端 context | `sslvpn context remote` | `service enable`，DNS `223.5.5.5` |

> ⚠ **三个已修正的写法错误**（对照 H3C 官方配置举例）：
> 1. 地址池须在**系统视图**用 `sslvpn ip address-pool` 创建，context 内是**引用**（原文档把两者混写成一条）；
> 2. 下发给客户端的路由须经 **`ip-route-list` + `include`** 再由 `ip-tunnel access-route` 引用（原文档直接写网段）；
> 3. **`filter ip-tunnel acl` 是必配项**（原文档漏了，等于不做访问控制）。

---

## 3. 安全策略（阶段 8 ⑧⑨）

```
# ⑧ 安全域
security-zone name VPN
 import interface SSLVPN-AC 1
quit

# ⑨ 安全策略
security-policy ip
 rule name VPN-Login
  source-zone Untrust
  destination-zone Local
  destination-ip-host 60.191.84.165
  service-port tcp destination eq 4430
  action pass
 quit
 rule name VPN-to-MGMT-443
  source-zone VPN
  destination-zone Trust
  source-ip-subnet 192.168.99.0 255.255.255.0
  destination-ip-subnet 192.168.10.0 255.255.255.0
  service-port tcp destination eq 443
  action pass
 quit
 rule name VPN-to-MGMT-22
  source-zone VPN
  destination-zone Trust
  source-ip-subnet 192.168.99.0 255.255.255.0
  destination-ip-subnet 192.168.10.0 255.255.255.0
  service-port tcp destination eq 22
  action pass
 quit
 rule name VPN-to-MGMT-IPMI
  source-zone VPN
  destination-zone Trust
  source-ip-subnet 192.168.99.0 255.255.255.0
  destination-ip-subnet 192.168.10.0 255.255.255.0
  service-port udp destination eq 623
  action pass
 quit
 rule name VPN-to-BIZ
  source-zone VPN
  destination-zone Trust
  source-ip-subnet 192.168.99.0 255.255.255.0
  destination-ip-subnet 192.168.11.0 255.255.255.0
  action pass
 quit
quit
save force
```

### 3.1 ⚠⚠ 两道过滤，排障必须先看第一道

| 顺序 | 过滤层 | 依据 | 特性 |
|---|---|---|---|
| **第 1 道** | **`filter ip-tunnel acl 3999`** | **目的 IP** | SSL VPN 网关层，**比安全策略更靠前**。客户端**只能**访问 ACL 里 permit 的目的地址，其余一律被网关丢弃（末尾隐含 deny） |
| 第 2 道 | 安全策略 | 安全域 + 服务 | 按 `source-zone` / `destination-zone` / 端口过滤 |

> **安全策略放行了也没用，只要目的 IP 不在这条 ACL 内就出不去** —— 2026-09-24 现场在这里卡得最久。
> 例：要让运维经 VPN 访问设备自身（隧道网关 `192.168.99.254`），**必须在 ACL 3999 里放行 `192.168.99.0/24`**。

---

## 4. 回程路由（两侧各一条，**缺了就是单向不通**）

```
# S6880（192.168.10.254）
system-view
ip route-static 192.168.99.0 24 192.168.254.1
save force

# S5120V3（192.168.10.251）—— 让管理交换机本身也能被客户访问
system-view
ip route-static 192.168.99.0 24 192.168.10.254
save force
```

| 设备 | 路由 | 说明 |
|---|---|---|
| S6880 | `192.168.99.0/24 → 192.168.254.1` | 让内网把回程交给防火墙 |
| S5120V3 | `192.168.99.0/24 → 192.168.10.254` | 该型号官方规格**支持静态路由**，不是纯二层，故本机也能被访问 |
| 防火墙 | `ip route-static 192.168.0.0 16 192.168.254.2` | 已覆盖该网段，无需新增 |
| **服务器侧** | **无需配置** | 业务口网关是 S6880（`192.168.11.254`）、BMC 网关是 `192.168.10.254`，回包都经 S6880 转给防火墙 |

---

## 5. 客户账号登记与日常管理

### 5.1 当前账号

| 用户名 | 类型 | 绑定策略组 | 用途 | 备注 |
|---|---|---|---|---|
| `ps` | `class network` | `customer` | 客户接入 | 口令见《客户使用说明.md》 |
| `cust01` | `class network` | `customer` | 客户接入（首位） | ⚠ **待改**：口令原与设备 admin 相同 |
| `cust02`… | 同上 | `customer` | 按客户数逐个新增 | 每台电脑一个账号，便于单独停用与追溯 |

### 5.2 增 / 改 / 停 / 删 / 查

> 先经 VPN 登录防火墙：`ssh admin@192.168.99.254`
> （Xshell / MobaXterm；Windows 自带 ssh 需加 `-o HostKeyAlgorithms=+ssh-rsa`）

| 操作 | 命令（`system-view` 后） |
|---|---|
| **新增** | `local-user <名> class network` → `password simple <专属口令>` → `service-type sslvpn` → `authorization-attribute sslvpn-policy-group customer` → `authorization-attribute user-role network-operator` → `quit` → `save force` |
| **改口令** | `local-user <名> class network` → `password simple <新口令>` → `quit` → `save force` |
| **临时停用** | `local-user <名> class network` → `state block` → `quit` → `save force` |
| **恢复使用** | 同上，改为 `state active` |
| **彻底删除** | 用户视图 `<H3C>` 下：`undo local-user <名>` → `save force` |
| **复核单个** | `display local-user user-name <名> class network` |
| **列出全部** | `display local-user` |
| **看在线客户** | `display sslvpn session` |

> ⚠ **五个必配项缺一不可**，尤其 **`service-type sslvpn`**（漏了这行用户登不上 VPN）；**每次都要 `save force`**。
> ⚠ **客户口令必须与设备 admin 口令无关** —— admin 口令能登防火墙本机，客户拿到等于拿到设备管理权。
> 💡 客户离职/暂停用 `state block`（留痕、可恢复），确定不再用才 `undo local-user`。
> ⚠ **`display local-user user-name X` 后面必须跟 `class`**，否则报 `Incomplete command`；本方案客户账号是 `class network`，要写全。
> **本机密码策略实测**：只启用了 `Password complexity: username checking`（口令不得含用户名），**未启用最小长度/老化限制**。

---

## 6. 客户端安装与连接（Windows）

1. **下载**：浏览器打开 `https://60.191.84.165:4430` → 登录页点 **「IP接入客户端」** → 下载 **iNode** 安装包。
   ⚠ **必须从该页面下载** —— 页面上的安装包是按本网关定制的；从官网随便下的可能识别不了网关，连上就被踢。
2. **安装**：双击安装，一路默认（需**管理员权限**）。
3. **连接**：打开 iNode → 新建 VPN 连接 → 服务器填 **`60.191.84.165:4430`**（⚠ **必须带端口**，只填 IP 会打到默认 443 上）→ 用户名/口令用分配的账号 → 连接。
4. **确认成功**：电脑网络连接里多出一块**虚拟网卡**（`iNode VPN Virtual NIC`），地址为 `192.168.99.x`。
5. **本机自检**（不依赖内网链路，当场可做）：

```powershell
ipconfig /all                       # 多出 iNode VPN Virtual NIC，地址 192.168.99.x，DNS 223.5.5.5
route print | findstr 192.168       # 应有两行：
                                    #   192.168.10.0  255.255.255.0  192.168.99.254
                                    #   192.168.11.0  255.255.255.0  192.168.99.254
```
   - 这两条路由即 VPN 下发的内网路由，**有它们就说明策略组 / route-list / access-route 全链生效**；
   - 默认路由 `0.0.0.0/0` **不应**被 VPN 接管 ⇒ 属**分流模式**，客户端平时上网不受影响。

> **登录后那个 Web 门户里为什么是空的**（书签 / TCP 资源 / 应用程序 / 文档中心都无内容）：本方案启用的是 **IP 接入**（网络扩展），没有配 Web 书签与 TCP 资源 ⇒ **空栏目属正常**。实际使用方式就是"建立 IP 隧道后按内网地址访问"。

---

## 7. 连上之后怎么用（各目标入口 · 2026-09-24 实测）

| 目标 | 从 VPN 用哪个地址 | 靠哪条策略放行 |
|---|---|---|
| **防火墙** | **`192.168.99.254`**（隧道网关 = 设备自身，同 VPN 域） | 无需额外策略；仅需 ACL 3999 放行 `192.168.99.0/24` |
| S6880（业务交换机） | `ssh admin@192.168.10.254` | `VPN-to-MGMT-22` |
| S5120V3（管理交换机） | `ssh admin@192.168.10.251` | `VPN-to-MGMT-22` |
| MQM8790（算力交换机） | `192.168.10.252` | `VPN-to-MGMT-22` |
| 服务器 OS | `ssh ps@192.168.11.11`–`.20` | `VPN-to-BIZ` |
| 服务器 BMC | `https://192.168.10.11`–`.20`（**必须带 `https://`**） | `VPN-to-MGMT-443` |

> ⚠ 防火墙**不能**用管理口地址 `192.168.10.253` —— 管理口属 `Management` 域，不在放行范围内；改用隧道网关 `.99.254` 即可。
> ⚠ 验证连通性用 **`Test-NetConnection <地址> -Port 22`** 或直接 SSH，**不要用 ping**（上述策略均未放行 ICMP）。
> ⚠ **新版 OpenSSH 会报 `no matching host key type found. Their offer: ssh-rsa`** —— Windows 自带 OpenSSH 默认禁用 `ssh-rsa`（SHA-1），而 H3C 只提供该算法。解决：① 用 Xshell / PuTTY / MobaXterm（最省事）；② 或加 `-o HostKeyAlgorithms=+ssh-rsa`。
> ⚠ 这条报错**说明网络已通、SSH 服务已应答**（对比 `Connection timed out` 是包根本没到）。

---

## 8. 安全建议（四条）

| # | 建议 | 做法 |
|---|---|---|
| 1 | **限源** | 客户在深圳的出口 IP 固定时，给 `VPN-Login` 加 `source-ip-subnet <客户出口IP> 255.255.255.0`，把 4430 从"对全网开放"收成"只对客户出口开放" |
| 2 | **按端口收紧业务网** | `VPN-to-BIZ` 目前是**全端口放行**（客户"用机"端口无法穷举）。若只需 SSH，改成 `service-port tcp destination eq 22`；若只需其中几台机器，把 ACL 3999 的第 2 条改成具体节点地址（如 `192.168.11.13 0.0.0.0`）—— **ACL 改动即时生效、客户端无需重连** |
| 3 | **留审计** | 每客户独立账号（本方案已如此）；在线会话 `display sslvpn session`；停用某人直接 `state block` / `undo local-user` |
| 4 | **加设备自身运维通道时先分离口令** | 需要"人在外地也能改防火墙"时补 `VPN-to-Local-SSH`（只放 22、不放 443）。⚠ **加之前必须先把客户账号口令与设备 admin 口令分离** —— 该策略对所有 VPN 用户生效 |

> **业务网全端口放行是安全上最需要关注的一条**：等于把算力集群对客户侧开放。上面第 2 条是最直接的收紧手段。

---

## 9. 验证

### 9.1 防火墙侧

```
display sslvpn gateway                  # 期望 Operation state: Up、IP: 60.191.84.165  Port: 4430
display sslvpn context                  # 期望 Operation state: Up、Password authentication: Enabled、关联网关 remotegw
display local-user                      # 列出全部本地用户
display local-user user-name cust01 class network
display sslvpn session                  # 有客户在线时会话可见
display license                         # 确认 15 个并发授权
```

> `display sslvpn context` 输出里的 **`Default policy group: Not configured` 属正常** —— 本方案是给用户**显式绑定**策略组，不依赖默认策略组。
> ⚠ `display sslvpn user` 本固件**不存在**（报 Unrecognized）。

### 9.2 入口可达性（不依赖内网链路，随时可做）

用手机（走蜂窝数据、**别连 Wi-Fi**）打开 `https://60.191.84.165:4430`，出现 SSL VPN 登录页即说明 **4430 对外入口已通**。

### 9.3 客户端侧（让客户做，你在防火墙上配合看会话）

| # | 检查 | 通过标准 |
|---|---|---|
| 1 | iNode 连接 | 显示「建立VPN隧道成功」并给出 IPv4 地址 |
| 2 | 本机自检 | `ipconfig` 有 `192.168.99.x`；`route print` 有两条内网路由；默认路由未被接管 |
| 3 | 管理网 | `ping 192.168.10.11` 通；`Test-NetConnection 192.168.10.251 -Port 22` → `TcpTestSucceeded: True`（**实测通过**） |
| 4 | BMC | 浏览器 `https://192.168.10.11` 能开登录页 |
| 5 | 业务网 | `ssh ps@192.168.11.13` 通（**实测通过**） |

> ⚠ **前置依赖**：`Vlan-if10`（需 S5120V3 上行）与 `Vlan-if11`（需服务器接入）**必须先 up**。否则隧道能建立、客户端也能拿到 `192.168.99.x`，但 ping 任何内网地址都不通 —— 那是**物理链路的问题，不是 VPN 的问题**。判断 VPN 是否成功，看第 1、2 步，不看 ping 内网。

---

## 10. 关键参数速查

| 参数 | 值 |
|---|---|
| 登录入口 | `https://60.191.84.165:4430` |
| 客户端软件 | iNode（从上述登录页下载，**必须带端口**） |
| 地址池 | `vpnpool` = 192.168.99.1–192.168.99.100 |
| 隧道接口 | `interface SSLVPN-AC 1` = 192.168.99.254/24 |
| SSL 网关 | `remotegw`，挂 `.165:4430` |
| context | `remote` |
| 访问控制 | `acl advanced 3999`（目的 IP 白名单） |
| 下发路由 | `ip-route-list intra`：192.168.10.0/24 + 192.168.11.0/24 |
| 策略组 | `customer` |
| 客户账号 | `class network` + `service-type sslvpn` + `sslvpn-policy-group customer` + `user-role network-operator` |
| 安全策略 | `VPN-Login` + `VPN-to-MGMT-443/22/IPMI` + `VPN-to-BIZ` |
| 回程路由 | S6880 → 192.168.254.1；S5120V3 → 192.168.10.254 |
| 授权 | 出厂自带 **15 并发** |
| 分流模式 | 只下发内网路由，不接管默认路由 |

---

## 11. 本篇相关坑（详见 [11-附录B](./11-附录B-现场坑与排障.md)）

1. **两层过滤顺序搞反** —— ACL 3999 比安全策略更靠前，不放行目的 IP 就出不去（卡最久的一条）。
2. **`sslvpn enable` 报 Unrecognized** —— 本机固件没有这条命令，SSL VPN 默认可用，跳过即可。
3. **`display sslvpn user` 不存在**；`display local-user user-name X` 后面必须跟 `class`。
4. **`filter ip-tunnel acl` 漏配** = 不做访问控制。
5. **`service-type sslvpn` 漏配** = 用户登不上。
6. **iNode 网关栏不带端口** = 连不上（默认打到 443）。
7. **iNode 安装包必须用登录页那份** —— 别处下载的可能识别不了网关。
8. **只放行特定 TCP 端口的策略不能用 `ping` 验证** —— ICMP 没被放行，ping 必然超时，**不能据此判断策略无效**。
9. **新版 OpenSSH 的 `ssh-rsa` 报错** —— 换客户端或加 `-o HostKeyAlgorithms=+ssh-rsa`。
10. **客户端本地网段冲突** —— 客户电脑所在网络也用 `192.168.10.x` / `192.168.11.x` 时，VPN 路由会被本地路由抢走（表现为"能连上但 ping 不通"）。
11. **入口单点** —— 网关只挂电信 `.165`，电信线故障时 VPN 入口随之中断（集群仍能出网）。要入口备份，在联通侧再建一个 `sslvpn gateway`（`101.71.15.156 port 4430`）绑同一 context（多网关支持以实测为准）。
12. **BMC 页面必须带 `https://`** —— 且从 VPN 访问时只放行 443、未放行 80。
