# Linux_iptables_详解

Linux iptables 详解

概念 · 原理 · 表链结构 · 数据包流程 · 实战应用

基于 Netfilter 框架的 Linux 内核数据包过滤与网络地址转换技术文档

2026 年 9 月


## 目录


## 1. iptables 概述与网络架构定位


### 1.1 什么是 iptables

iptables 是 Linux 系统中用于配置内核 IPv4 数据包过滤 与 网络地址转换 规则的用户空间命令行工具。它为系统管理员提供了一套简洁的接口，通过在既定的规则表（Table）和规则链（Chain）中增删规则，来控制数据包的接收、转发、修改与丢弃行为，从而构建防火墙、实现 NAT、完成端口映射等功能。

需要特别强调一个关键认知：iptables 本身并不执行任何过滤动作。它只是一个“配置前端”，负责把用户编写的规则翻译并下发到内核中的 Netfilter 框架；真正对网络数据包进行匹配、修改、丢弃或转发的，是内核里的 Netfilter 子系统。


### 1.2 在 Linux 网络架构中的定位

Linux 网络协议栈可分为若干层次，iptables 在整个架构中的位置可以用下面的层次关系来理解：

┌──────────────────────────────────────────────────────────┐

│                用户空间 (User Space)                       │

│   应用程序 / 浏览器 / 服务进程                              │

│   ┌────────────────────────────────────────────┐          │

│   │  iptables 命令   (配置与管理规则)            │          │

│   └───────────────────┬────────────────────────┘          │

└───────────────────────┼────────────────────────────────────┘

setsockopt() / 系统调用 (下发规则)   │

┌───────────────────────┼────────────────────────────────────┐

│  ┌────────────────────▼────────────────────────┐           │

│  │     Netfilter 框架 (内核中的钩子 hooks)      │           │

│  │     PREROUTING / INPUT / FORWARD /          │           │

│  │     OUTPUT / POSTROUTING                    │           │

│  └────────────────────┬────────────────────────┘           │

│   TCP/IP 协议栈实现 (ip_rcv, ip_forward ...)                │

│   网卡驱动 / 网络设备层                                     │

│             内核空间 (Kernel Space)                         │

└─────────────────────────────────────────────────────────────┘

从定位上看，iptables 处于 用户空间的管理层，是人与内核过滤机制之间的桥梁。它工作在 网络层（IP 层），主要基于 IP 报文的源地址、目的地址、协议类型、端口号等信息进行判断，属于典型的包过滤（Packet Filtering）技术。


### 1.3 核心作用

包过滤与访问控制：依据规则允许或拒绝特定流量，构建主机防火墙或网关防火墙。

网络地址转换（NAT）：实现源地址转换（SNAT/MASQUERADE）使内网主机共享公网出口，或通过目的地址转换（DNAT）把外部请求转发到内部服务器。

端口转发与负载分发：将到达某端口的流量重定向到其他主机或端口，配合统计模块实现简单负载均衡。

流量控制与限速：利用 limit、connlimit、recent 等模块对连接数、请求速率进行限制，缓解 DoS 攻击。

报文修改：通过 mangle 表修改 TTL、TOS、MARK 等字段，配合策略路由与 QoS。


## 2. Netfilter 框架与 iptables 的关系


### 2.1 Netfilter 是什么

Netfilter 是 Linux 2.4 内核引入的数据包处理框架。它在内核网络协议栈的关键位置埋入了一系列 钩子函数（hook），当数据包流经这些位置时，内核会自动调用注册在钩子上的处理函数。由于钩子嵌在协议栈内部，数据包无需离开内核空间即可完成全部处理，因此性能极高，同时保持了良好的模块化与可扩展性。

Netfilter 在内核中定义了五个标准的钩子点（对应五条链），任何模块都可以在这些点上注册回调，从而参与到数据包处理流程当中。


### 2.2 iptables 与 Netfilter 的分工

二者的关系可以概括为“前端与后端”或“配置者与执行者”：


| 对比维度 | Netfilter | iptables |
|---|---|---|
| 所在空间 | 内核空间（Kernel Space） | 用户空间（User Space） |
| 角色定位 | 数据包处理框架、执行引擎 | 规则配置工具、管理前端 |
| 核心职责 | 提供钩子、按规则匹配并执行动作 | 解析命令行、增删改查规则 |
| 表现形式 | 内核模块与钩子函数 | iptables / ip6tables 命令 |
| 运行时机 | 每个数据包经过时实时执行 | 管理员手动或脚本触发执行 |
| 依赖关系 | 独立存在，即使无 iptables 也运行 | 依赖内核中的 Netfilter 支持 |

因此，当我们执行一条 iptables -A INPUT -p tcp --dport 22 -j ACCEPT 时，实际发生的过程是：iptables 命令解析参数 → 通过系统调用（早期为 setsockopt，现在多经由 nf_tables / legacy API）把规则结构体传递给内核 → 内核 Netfilter 把规则挂载到 INPUT 链上 → 此后每个进入本机的 TCP 报文到达 INPUT 钩子点时，都会被这条规则匹配。


### 2.3 相关工具家族

随着内核演进，围绕 Netfilter 出现了多个用户态工具，它们操作的是同一套内核框架的不同接口：

iptables：IPv4 规则管理工具，最经典、最广泛使用。

ip6tables：IPv6 规则管理工具，语法与 iptables 基本一致。

arptables：ARP 报文规则管理工具。

ebtables：数据链路层（以太网帧）过滤工具。

nftables：新一代替代方案（nft 命令），统一了上述工具的功能，内核 Linux 3.13+ 提供，语法更简洁、性能更好，是未来趋势。


## 3. 表（Tables）与链（Chains）的结构

iptables 的核心组织模型是“表 + 链 + 规则”三层结构：

规则（Rule）：最基本单位，由“匹配条件 + 目标动作”组成，例如“源 IP 为 10.0.0.5 就丢弃”。

链（Chain）：一组规则的顺序集合，数据包会自上而下逐条匹配，命中即执行动作。

表（Table）：按功能划分的链的集合，每张表只负责一类任务（过滤、NAT、修改、连接跟踪）。


### 3.1 四种主要表（Tables）


| 表名 | 核心功能 | 包含的链 | 典型用途 |
|---|---|---|---|
| filter | 数据包过滤（默认表） | INPUT, FORWARD, OUTPUT | 防火墙、访问控制、允许/拒绝流量 |
| nat | 网络地址转换 | PREROUTING, OUTPUT, POSTROUTING | SNAT、DNAT、MASQUERADE、端口映射 |
| mangle | 修改报文头部字段 | 全部五条链 | 改 TTL/TOS/MARK、策略路由、QoS |
| raw | 连接跟踪前的原始处理 | PREROUTING, OUTPUT | 设置 NOTRACK，跳过 conntrack 提升性能 |

优先级顺序：当数据包到达同一钩子点时，表的处理顺序从高到低为 raw → mangle → nat → filter（在 PREROUTING 中 nat 优先于 filter，因为 filter 不参与 PREROUTING）。另有 security 表用于 SELinux 等安全模块，优先级在 filter 之后。


#### filter 表

最常用、也是默认操作的表，不指定 -t 参数时默认即对 filter 表操作。它决定数据包是被接受还是被拒绝/丢弃。


#### nat 表

负责地址转换。需注意：只有连接的第一个数据包会真正经过 nat 表，后续同连接的报文由内核连接跟踪（conntrack）直接按已建立的映射处理，因此 nat 表的规则不会对每个包重复执行。


#### mangle 表

用于修改报文内容，如调整 TTL、设置 TOS/DSCP 优先级、打 MARK 标记（供 ip rule 策略路由使用）等。


#### raw 表

优先级最高，主要用于在连接跟踪之前对特定流量使用 NOTRACK 目标，避免 conntrack 记录，降低大流量场景下的内存与 CPU 开销。


### 3.2 五条链（Chains）及其用途


| 链名 | 触发时机（数据包所处阶段） | 主要用例 |
|---|---|---|
| PREROUTING | 数据包刚到达网卡、进入协议栈，尚未做路由判断 | DNAT、端口映射、修改目的地址 |
| INPUT | 路由判断后，确认目标是本机的数据包 | 保护本机服务（防火墙核心） |
| FORWARD | 路由判断后，需要转发给其他主机的数据包 | 网关/路由器防火墙、转发控制 |
| OUTPUT | 本机进程产生、即将发出的数据包 | 限制本机对外访问 |
| POSTROUTING | 数据包即将离开网卡、发出之前 | SNAT、MASQUERADE（共享上网） |

可以这样直观记忆：PREROUTING → INPUT → 本机进程 → OUTPUT → POSTROUTING 是数据包进出本机的路径；而 PREROUTING → FORWARD → POSTROUTING 是需要转发数据包所走的路径。


### 3.3 表与链的对应关系矩阵


| 链 \ 表 | raw | mangle | nat | filter |
|---|---|---|---|---|
| PREROUTING | ✔ | ✔ | ✔ | — |
| INPUT | — | ✔ | — | ✔ |
| FORWARD | — | ✔ | — | ✔ |
| OUTPUT | ✔ | ✔ | ✔ | ✔ |
| POSTROUTING | — | ✔ | ✔ | — |

说明：“✔”表示该表在该链上存在可用的钩子；“—”表示该组合不存在。这也解释了为什么在 PREROUTING 上无法写 filter 规则、在 FORWARD 上无法写 nat 规则。


## 4. 规则匹配条件与目标动作

一条 iptables 规则由两部分组成：匹配条件（matches） 与 目标动作（target）。

iptables [-t 表名] 命令 [链名] [匹配条件...] -j 目标动作 [目标选项...]


### 4.1 通用匹配条件


| 参数 | 含义 | 示例 |
|---|---|---|
| -p, --protocol | 指定协议：tcp/udp/icmp/all | iptables -A INPUT -p tcp |
| -s, --source | 源地址或网段 | -s 192.168.1.0/24 |
| -d, --destination | 目的地址或网段 | -d 10.0.0.1 |
| -i, --in-interface | 入站网卡接口 | -i eth0 |
| -o, --out-interface | 出站网卡接口 | -o eth1 |
| -f, --fragment | 匹配分片报文的第二片及之后 | -f |


### 4.2 协议扩展匹配

当使用 -p tcp / -p udp / -p icmp 后，可通过扩展模块匹配端口、标志位、ICMP 类型等：

--sport / --dport：源端口 / 目的端口，可指定范围，如 --dport 1024:65535

--tcp-flags：匹配 TCP 标志位，如 --tcp-flags SYN,RST SYN

--syn：匹配 TCP 连接请求报文（等价于 SYN 置位且 ACK 未置位）

--icmp-type：匹配 ICMP 类型，如 --icmp-type echo-request


### 4.3 常用扩展模块匹配


| 模块 (-m) | 用途 | 常用参数 |
|---|---|---|
| multiport | 一次匹配多个不连续端口 | --dports 22,80,443 |
| iprange | 匹配 IP 地址范围 | --src-range 10.0.0.10-10.0.0.20 |
| mac | 匹配源 MAC 地址 | --mac-source 00:11:22:33:44:55 |
| state | 匹配连接状态 | --state NEW,ESTABLISHED,RELATED |
| conntrack | 更强大的连接跟踪匹配 | --ctstate NEW --ctproto tcp |
| limit | 按速率限制匹配频率 | --limit 10/min --limit-burst 20 |
| connlimit | 限制并发连接数 | --connlimit-above 50 |
| recent | 基于最近访问记录匹配（防爆破） | --update --seconds 60 --hitcount 5 |
| time | 按时间段匹配 | --timestart 09:00 --timestop 18:00 |
| string | 按报文载荷字符串匹配 | --string "password" --algo bm |
| owner | 按创建报文的本地用户匹配（仅 OUTPUT） | --uid-owner 1000 |
| comment | 为规则添加备注，便于维护 | --comment "allow ssh" |

关于连接状态（state/conntrack）：这是 iptables 最强大的特性之一，基于连接跟踪机制，把报文归类为几种状态：

NEW：新建连接的第一个报文（如 TCP 的 SYN）。

ESTABLISHED：属于已建立连接、双向通信的报文。

RELATED：与已有连接相关联的新连接，如 FTP 数据通道、ICMP 差错报文。

INVALID：无法识别或状态异常的报文，通常直接丢弃。

典型用法：只放行 ESTABLISHED,RELATED 与特定入口的 NEW，即可实现“状态防火墙”，既安全又高效。


### 4.4 目标动作（Targets）

目标动作通过 -j（jump）指定，分为两大类：


#### 基本动作（终止型）


| 动作 | 作用 | 说明 |
|---|---|---|
| ACCEPT | 接受数据包，放行 | 不再匹配本链后续规则 |
| DROP | 丢弃数据包 | 不返回任何响应，对端表现为超时 |
| REJECT | 拒绝数据包 | 返回 ICMP 错误（默认 port-unreachable） |
| RETURN | 返回上一级链继续处理 | 在当前链中终止匹配并退回调用链 |
| LOG | 记录日志到内核日志 | 非终止动作，记录后继续匹配后续规则 |


#### NAT 与特殊动作


| 动作 | 所属表 | 作用 |
|---|---|---|
| SNAT | nat | 修改源地址（POSTROUTING），需指定 --to-source，适合固定公网 IP |
| MASQUERADE | nat | 源地址伪装，自动使用出口网卡 IP（POSTROUTING），适合动态 IP（如拨号） |
| DNAT | nat | 修改目的地址（PREROUTING/OUTPUT），需指定 --to-destination，用于端口映射 |
| REDIRECT | nat | 把报文重定向到本机其他端口（透明代理常用） |
| MARK | mangle | 给报文打标记，供策略路由或 QoS 使用 |
| TTL/TOS | mangle | 修改报文 TTL / 服务类型字段 |
| NOTRACK | raw | 跳过连接跟踪处理 |
| 自定义链 | 任意 | 用 -N 创建链名，通过 -j 跳转，实现规则复用与模块化 |


#### 自定义链示例

# 创建自定义链

iptables -N WEB_FILTER

# 在自定义链中添加规则

iptables -A WEB_FILTER -p tcp --dport 80 -j ACCEPT

iptables -A WEB_FILTER -p tcp --dport 443 -j ACCEPT

# 在 INPUT 链中跳转到自定义链

iptables -A INPUT -j WEB_FILTER


## 5. 数据包在内核中的过滤与转发流程

理解数据包流经各钩子的顺序，是掌握 iptables 的关键。下图展示了数据包进入系统后的完整路径，以及各阶段会触发哪些表的链。


### 5.1 完整流程图

┌──────────────┐

数据包到达 ───▶ │   网卡网卡    │

└──────┬───────┘

▼

┌───────────────────┐

│  PREROUTING 链     │  raw → mangle → nat(DNAT)

└─────────┬─────────┘

▼

┌───────────────────┐

│   路由判断         │  目的地址是本机还是其他主机?

└────┬─────────┬────┘

┌───────────┘         └───────────┐

▼(目标为本机)                  ▼(需转发)

┌───────────────────┐          ┌───────────────────┐

│   INPUT 链         │          │  FORWARD 链        │

│  mangle → filter   │          │  mangle → filter   │

└─────────┬─────────┘          └─────────┬─────────┘

▼                                ▼

┌───────────────────┐          ┌───────────────────┐

│  本机上层进程      │          │  POSTROUTING 链    │

│  (TCP/UDP/应用)    │          │  mangle → nat(SNAT)│

└─────────┬─────────┘          └─────────┬─────────┘

│ 本机发出                      ▼

▼                         出网卡发送 ▶

┌───────────────────┐

│  路由判断          │

└─────────┬─────────┘

▼

┌───────────────────┐

│   OUTPUT 链        │  raw → mangle → nat → filter

└─────────┬─────────┘

▼

┌───────────────────┐

│ POSTROUTING 链     │  mangle → nat(SNAT)

└─────────┬─────────┘

▼

出网卡发送 ▶


### 5.2 三条典型路径小结


| 数据包走向 | 经过的链（按顺序） | 典型表处理 |
|---|---|---|
| 发送给本机的包 | PREROUTING → INPUT | nat(DNAT) → filter |
| 需转发的包 | PREROUTING → FORWARD → POSTROUTING | nat(DNAT) → filter → nat(SNAT) |
| 本机发出的包 | OUTPUT → POSTROUTING | filter → nat(SNAT) |


### 5.3 关键机制说明

路由判断发生两次：第一次在 PREROUTING 之后，决定报文是入本机还是被转发；第二次在本机发出报文时，决定从哪个网卡出。

DNAT 只生效一次：nat 表只在连接建立时起作用，后续报文按 conntrack 记录处理。

过滤可发生在多个位置：INPUT 过滤发往本机的流量，FORWARD 过滤穿越流量，OUTPUT 过滤本机发出的流量。

匹配即停止：同一链内规则自上而下匹配，一旦命中目标动作（ACCEPT/DROP 等）即停止后续匹配；非终止动作（如 LOG）则继续匹配。


## 6. 典型实际应用场景


### 6.1 场景一：主机防火墙（保护本机服务）

通过 INPUT/OUTPUT 链限制对本机服务的访问，只放行必要端口，并采用状态检测。

# 清空现有规则

iptables -F

iptables -X

# 允许本地环回

iptables -A INPUT -i lo -j ACCEPT

iptables -A OUTPUT -o lo -j ACCEPT

# 放行已建立连接和相关流量（状态检测，提高效率）

iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# 放行 SSH(22)、HTTP(80)、HTTPS(443)

iptables -A INPUT -p tcp --dport 22  -j ACCEPT

iptables -A INPUT -p tcp --dport 80  -j ACCEPT

iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 允许 ping（可选）

iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT

# 默认策略：丢弃所有其他入站流量

iptables -P INPUT   DROP

iptables -P FORWARD DROP

iptables -P OUTPUT  ACCEPT


### 6.2 场景二：NAT —— 内网共享上网（SNAT / MASQUERADE）

在充当网关的机器上开启转发，并将内网流量源地址转换为公网地址。

# 1) 开启内核 IP 转发

echo 1 > /proc/sys/net/ipv4/ip_forward

# 永久生效: 编辑 /etc/sysctl.conf 设置 net.ipv4.ip_forward = 1 后 sysctl -p

# 2) 源地址转换：固定公网 IP 用 SNAT

iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j SNAT --to-source 203.0.113.10

# 或使用 MASQUERADE（适合动态获取公网 IP 的场景，如 ADSL 拨号）

iptables -t nat -A POSTROUTING -s 192.168.1.0/24 -o eth0 -j MASQUERADE

# 3) 允许内网转发到外网

iptables -A FORWARD -s 192.168.1.0/24 -o eth0 -j ACCEPT

iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT


### 6.3 场景三：端口转发 / DNAT 端口映射

把公网某个端口的访问请求转发到内网服务器，常用于暴露内网服务。

# 将公网 80 端口请求转发到内网 Web 服务器 192.168.1.100:8080

iptables -t nat -A PREROUTING -p tcp --dport 80 \

-j DNAT --to-destination 192.168.1.100:8080

# 放行转发流量

iptables -A FORWARD -p tcp -d 192.168.1.100 --dport 8080 -j ACCEPT

# 本机自身访问时使用 OUTPUT 链的 DNAT（可选）

iptables -t nat -A OUTPUT -p tcp --dport 80 -j DNAT --to-destination 192.168.1.100:8080


### 6.4 场景四：流量控制与安全防护

利用限速、连接数限制、recent 等模块缓解暴力破解与 DoS 攻击。

# 限制 SSH 连接频率：60 秒内超过 4 次新连接则拉黑 300 秒

iptables -A INPUT -p tcp --dport 22 -m state --state NEW \

-m recent --set --name SSH

iptables -A INPUT -p tcp --dport 22 -m state --state NEW \

-m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP

# 限制单 IP 对 80 端口的并发连接不超过 20

iptables -A INPUT -p tcp --dport 80 -m connlimit --connlimit-above 20 -j REJECT

# 限制 ICMP 泛洪：每秒最多 10 个 ping 包

iptables -A INPUT -p icmp --icmp-type echo-request \

-m limit --limit 10/s --limit-burst 20 -j ACCEPT

# 丢弃状态非法的报文

iptables -A INPUT -m state --state INVALID -j DROP

# 记录并丢弃异常的 NULL 扫描报文

iptables -A INPUT -p tcp --tcp-flags ALL NONE -j LOG --log-prefix "NULL Scan: "

iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP


### 6.5 场景五：透明代理与端口重定向

通过 REDIRECT 将流量重定向到本机代理端口，实现透明代理（常见于网关与 Squid 结合）。

# 将所有经过本机的 80 端口流量重定向到本机 3128（Squid 代理）

iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 3128


### 6.6 场景六：修改报文标记配合策略路由

使用 mangle 表打标记，再配合 ip rule 实现多线路策略路由（如分流不同 ISP）。

# 给来自 192.168.2.0/24 的报文打上标记 1

iptables -t mangle -A PREROUTING -s 192.168.2.0/24 -j MARK --set-mark 1

# 使用 ip rule 让标记为 1 的报文走特定路由表

ip rule add fwmark 1 table 100

ip route add default via 10.0.0.2 dev eth1 table 100


## 7. 常用命令速查


### 7.1 规则管理命令


| 命令 | 含义 |
|---|---|
| -A 链名 | 在链末尾追加一条规则（Append） |
| -I 链名 [序号] | 在链首或指定位置插入规则（Insert） |
| -D 链名 序号 | 删除指定序号的规则（Delete） |
| -R 链名 序号 | 替换指定序号的规则（Replace） |
| -L [链名] | 列出规则（List），可加 -n 不解析 DNS、-v 显示计数、--line-numbers 显示序号 |
| -F [链名] | 清空所有或指定链的规则（Flush） |
| -N 链名 | 创建自定义链（New chain） |
| -X [链名] | 删除自定义链（需先清空且无引用） |
| -P 链名 动作 | 设置链的默认策略（Policy），如 -P INPUT DROP |
| -Z [链名] | 将计数器归零（Zero） |


### 7.2 查看与保存

# 以数字形式查看所有规则并显示计数器，带序号

iptables -n -v -L --line-numbers

# 查看 nat 表规则

iptables -t nat -L -n -v

# 导出规则（备份）

iptables-save > /etc/iptables/rules.v4

# 恢复规则

iptables-restore < /etc/iptables/rules.v4


### 7.3 持久化与最佳实践

持久化：iptables 规则默认重启后丢失。Debian/Ubuntu 可用 iptables-persistent；CentOS/RHEL 可用 iptables-services；也可将脚本写入 systemd 服务或 rc.local。

先放行再拒绝：配置远端服务器防火墙时，务必先允许 SSH 再设置 DROP 默认策略，避免把自己拒之门外。

善用状态检测：优先使用 ESTABLISHED,RELATED 放行回包，减少规则数量。

使用注释：对复杂规则使用 -m comment --comment "说明"，便于日后维护。

规则顺序很重要：iptables 自上而下匹配，宽泛规则应放在具体规则之后。


## 8. 局限性与技术演进

尽管 iptables 极其强大，但在现代高并发与复杂网络环境中也逐渐暴露出不足：

性能瓶颈：在规则数量巨大时，逐条匹配的线性遍历会造成性能下降。

语法复杂：多种表/链/扩展模块的组合对使用者门槛较高，规则难以维护。

功能分散：IPv4、IPv6、ARP、二层过滤分别由不同工具处理，缺乏统一。

补丁式扩展：大量功能通过扩展模块叠加，缺乏统一的数据模型。

为此，Linux 3.13 引入了 nftables 作为下一代框架，其核心优势包括：

统一工具 nft 同时管理 IPv4/IPv6/ARP/网桥，语法一致。

采用虚拟机式规则评估与集合（set/map），大幅提升大规模规则下的性能。

规则以原子事务方式提交，支持更好的并发与一致性。

提供了 iptables-nft 兼容层，可将 iptables 命令翻译为 nft 规则，便于平滑迁移。

小结：iptables 建立在 Netfilter 框架之上，通过“表—链—规则”的模型，在数据包流经内核协议栈的关键钩子点上实施过滤、修改与地址转换。它既是 Linux 服务器防火墙的事实标准，也是理解 Linux 网络安全机制的重要基石。掌握其原理与数据包流程，能够帮助我们在实际运维中构建安全、高效、可控的网络环境。
