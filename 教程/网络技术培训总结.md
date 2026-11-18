# 网络技术培训PPT总结

> 本文档对11个网络技术PPT进行逐一详细总结，**重点内容已加粗标注**。

---

## 1. 网络基础

### 内容概述
本PPT介绍了计算机网络的基础知识，包括网络拓扑结构、WAN与LAN的区别、常见网络设备、OSI与TCP/IP模型、数据封装过程以及交换机工作原理。

### 详细内容

#### 1.1 网络基础知识
- **网络拓扑结构**：介绍了星型拓扑和网型拓扑
  - **星型拓扑优点**：易于实现、易于网络扩展、易于故障排查
  - **星型拓扑缺点**：中心节点压力大、组网成本较高
  - **网型拓扑**：各个节点至少与其他两个节点相连，可靠性高但组网成本也高

- **WAN与LAN**：
  - **WAN（广域网）**：范围几十到几千千米，用于连接远距离计算机网络，典型应用是Internet
  - **LAN（局域网）**：范围约1km，用于连接较短距离内的计算机，典型应用是企业网、校园网

- **网络常见设备**：
  - 交换路由设备、网络安全设备（防火墙、VPN设备）、无线网络设备
  - **主要厂商**：Cisco（思科）、华为、H3C

#### 1.2 网络模型
- **OSI七层模型**：物理层、数据链路层、网络层、传输层、会话层、表示层、应用层
- **TCP/IP五层模型**：物理层、数据链路层、网络层、传输层、应用层
- **TCP/IP四层模型**：网络接口层、网络层、传输层、应用层

- **设备与层的对应关系**：
  - 物理层：网卡
  - 数据链路层：交换机
  - 网络层：路由器、防火墙
  - 传输层及以上：计算机

- **TCP/IP协议族组成**：
  - 应用层：HTTP、FTP、TFTP、SMTP、SNMP、DNS
  - 传输层：TCP、UDP
  - 网络层：ICMP、IGMP、IP、ARP、RARP

- **数据封装过程**：
  - 上层数据 → 添加TCP头部 → 添加IP头部 → 添加MAC头部 → 物理层传输

#### 1.3 交换机工作原理
- **以太网**：工作在数据链路层，数据单元为帧（Frame）
- **MAC地址**：48位地址，前24位为供应商标识，后24位为供应商对网卡的唯一编号
  - **重点**：目的地址第一位为0表示物理地址（单播），为1表示逻辑地址（组播）

- **Ethernet II帧格式**：
  - 前导码（7字节）+ 目的地址（6字节）+ 源地址（6字节）+ 类型（2字节）+ 数据（46~1500字节）+ 帧校验序列（4字节）
  - **类型字段**：用来标识上层协议类型，如0800H表示IP协议

- **交换机转发原理**：
  1. **初始状态**：MAC地址表为空
  2. **MAC地址学习**：记录源MAC地址与接口的对应关系
  3. **广播未知数据帧**：当目标MAC未知时，向所有端口广播
  4. **接收方回应**：目标主机回应，交换机学习到其MAC地址
  5. **实现单播通信**：后续通信直接转发到对应端口

- **交换机以太网接口工作模式**：
  - **单工**：两个数据站之间只能沿单一方向传输数据（如麦克风→扬声器）
  - **半双工**：两个数据站之间可以双向数据传输，但不能同时进行（如对讲机）
  - **全双工**：两个数据站之间可双向且同时进行数据传输（如打电话）

#### 1.4 IP地址及测试工具
- **IP地址组成**：网络部分（NETWORK）+ 主机部分（HOST）
- **IP地址分类**：A、B、C、D、E五类
  - **A类**：1~126，网络位8位，主机位24位
  - **B类**：128~191，网络位16位，主机位16位
  - **C类**：192~223，网络位24位，主机位8位

- **私有地址**（**重点**，不能在Internet上使用）：
  - A类：10.0.0.0 ~ 10.255.255.255
  - B类：172.16.0.0 ~ 172.31.255.255
  - C类：192.168.0.0 ~ 192.168.255.255

- **网络测试工具**：
  - **ipconfig**：查看本机IP配置
  - **ping**：测试网络连通性
    - ping 127.0.0.1：验证TCP/IP是否正确配置
    - ping本地IP：验证是否已正确添加到网络
    - ping默认网关：验证默认网关是否正常工作
    - ping远程主机：验证是否可以通过路由器通信
  - **Tracert**：跟踪路由路径

---

## 2. IP地址和子网掩码

### 内容概述
本PPT深入讲解IP地址和子网掩码的概念、分类、配置方法，以及通过配置IP地址实现路由器互通的实验。

### 详细内容

#### 2.1 IP地址概述与应用
- **IP地址的定义**：逻辑地址，用于在网络中标识设备
- **IP地址的分类**：A、B、C、D、E五类（同PPT1）
- **子网掩码**：用于区分IP地址中的网络部分和主机部分

#### 2.2 配置IP地址
- **设备配置准备工作**（**重点命令**）：
  ```
  en
  conf t
  no ip domain-lookup        // 禁用DNS查询
  line con 0
  logging synchronous        // 防止控制台消息打断输入
  exec-timeout 0 0           // 禁用空闲超时
  ```

- **路由器配置IP地址**：
  ```
  Router(config)#interface fastethernet 0/1
  Router(config-if)#ip address ip-address subnet-mask
  Router(config-if)#no shutdown
  ```

- **交换机配置IP地址**（在VLAN接口上配置）：
  ```
  Switch(config)#interface vlan 1
  Switch(config-if)#ip address ip-address subnet-mask
  Switch(config-if)#no shutdown
  ```

#### 2.3 查看接口状态
- **重点**：`show int f0/1` 查看接口状态
  - **物理层up，数据链路层up**：正常状态
  - **物理层down，数据链路层down**：物理连接故障
  - **物理层administratively down**：接口被管理员关闭

- **两端的接口状态都是"up"就一定能ping通吗？** 不一定，还需要IP地址配置正确。

#### 2.4 实验内容
- **实验一**：配置IP地址并验证连通性
  - 同一网段的两台主机可以互通
  - 不同网段的主机不能直接互通（需要路由器）

- **实验二**：配置路由器实现互通
  - 通过Console口配置路由器接口IP地址
  - 保存配置：`copy running-config startup-config`
  - 将配置文件备份到PC机

---

## 3. ICMP ARP协议原理

### 内容概述
本PPT讲解网络层协议，包括IP数据包格式、广播与广播域、ARP协议原理、ICMP协议原理及常用命令。

### 详细内容

#### 3.1 网络层功能与IP数据包格式
- **网络层功能**：
  - 定义基于IP协议的逻辑地址
  - 连接不同的媒介类型
  - **选择数据通过网络的最佳路径（路由）**

- **IP数据包格式**（**重点**，20字节首部）：
  - 版本（4位）：IPv4
  - 首部长度（4位）
  - 优先级与服务类型（8位）：提供3层的QoS
  - 总长度（16位）
  - 标识符（16位）、标志（3位）、段偏移量（13位）
  - **TTL（8位）**：生存时间，防止数据包无限循环
  - 协议号（8位）：标识上层协议
  - 首部校验和（16位）
  - 源地址（32位）、目标地址（32位）
  - 可选项、数据

#### 3.2 广播与广播域
- **广播**：将广播地址作为目的地址的数据帧
- **广播域**：网络中能接收到同一个广播所有节点的集合
- **MAC地址广播**：FF-FF-FF-FF-FF-FF
- **IP地址广播**：IP地址网段的广播地址

#### 3.3 ARP协议（**重点**）
- **ARP（Address Resolution Protocol，地址解析协议）**：将已知的IP地址解析成MAC地址

- **ARP工作原理**：
  1. PC1发送数据给PC2，查看ARP缓存没有PC2的MAC地址
  2. **PC1发送ARP请求消息（广播）**
  3. 所有主机收到ARP请求消息
  4. **PC2回复ARP应答（单播）**
  5. 其他主机丢弃
  6. PC1将PC2的MAC地址保存到缓存中，发送数据

- **ARP相关命令**：
  - Windows：`arp -a`（查看缓存）、`arp -d`（清除缓存）
  - Cisco：`show ip arp`（查看ARP缓存表）

#### 3.4 ICMP协议
- **ICMP（Internet Control Message Protocol）**：错误侦测与回馈机制，通过IP数据包封装
- **主要功能**：发送错误和控制消息
  - 例如：目标不可达、超时、重定向等

- **Ping命令**（**重点**）：
  - 基本格式：`ping [-t] [-l 字节数] [-a] [-i] IP_Address|target_name`
  - **-t**：一直不停地执行ping（按Ctrl+C中断）
  - **-a**：显示主机名称
  - **-l**：设定ping包的大小（测试通信质量）

- **Ping返回信息分析**：
  - 连通后的应答：正常
  - 请求超时：可能未正确配置网关等参数
  - 无法访问目标主机：路由问题

---

## 4. 了解路由器组成及产品

### 内容概述
本PPT介绍路由器的硬件组成、启动过程、IOS系统、密码设置与恢复、远程管理以及IOS备份升级。

### 详细内容

#### 4.1 路由器硬件概述
- **路由器内部组件**：CPU、RAM、ROM、Flash、NVRAM、接口等
- **Cisco路由产品体系**：了解不同系列路由器的应用场景

#### 4.2 路由器启动过程
- 启动过程：加电自检 → 加载IOS → 加载配置文件
- **配置寄存器**：控制启动行为，默认0x2102

#### 4.3 密码设置与恢复（**重点**）
- **密码设置**：
  ```
  enable password cisco0        // 明文特权密码
  enable secret cisco           // 密文特权密码（优先级更高）
  line console 0
  password benet
  login                         // 启用登录验证
  ```

- **加密明文密码**：
  ```
  service password-encryption   // 加密所有明文密码
  ```

- **路由器密码恢复思路**（**重点**）：
  1. 修改配置寄存器值为0x2142，启动不加载配置文件
  2. 正常启动后，用startup-config覆盖running-config
  3. 修改密码，恢复配置寄存器值为0x2102
  ```
  rommon1> confreg 0x2142
  rommon2> reset
  Router# copy startup-config running-config
  Router(config)# enable secret cisco
  Router(config)# config-register 0x2102
  ```

- **交换机密码恢复思路**（**重点**）：
  1. 拔掉电源，插上电源同时按住MODE键
  2. 出现"switch:"提示松开按键，初始化Flash
  3. 将config.text文件改成config.old
  4. 启动交换机后，把配置文件改回来
  ```
  switch: flash_init
  switch: rename flash:config.text flash:config.old
  switch: boot
  switch# rename flash:config.old flash:config.text
  switch# copy flash:config.text system:running-config
  ```

#### 4.4 远程管理
- **配置管理IP地址**：
  - 路由器：直接在接口配置IP
  - 交换机：在VLAN接口配置IP

- **配置VTY密码**（Telnet/SSH远程登录）：
  ```
  line vty 0 4
  password cisco
  login
  ```

- **配置默认网关**（交换机需要配置才能跨网段管理）：
  ```
  Switch(config)# ip default-gateway ip-address
  ```

- **远程管理必须配置**：登录密码和特权密码

#### 4.5 IOS管理
- **IOS命名规则**：AAAAA-BBBB-CC-DDDD.EE
  - AAAAA：适用硬件平台
  - BBBB：特性集
  - CC：运行方式和压缩格式
  - DDDD：软件版本
  - EE：文件后缀

- **IOS备份与升级**：
  - 通过TFTP服务器升级：`copy tftp flash`
  - 备份IOS到TFTP：`copy flash tftp`

---

## 5. VLAN的原理及配置

### 内容概述
本PPT讲解VLAN（虚拟局域网）的原理、优势、配置方法，以及Trunk原理和以太网通道配置。

### 详细内容

#### 5.1 VLAN概述
- **VLAN（Virtual Local Area Network，虚拟局域网）**：将物理网络逻辑划分为多个广播域
- **VLAN的优势**：
  - 控制广播风暴
  - 增强网络安全性
  - 简化网络管理
  - 灵活的分组

#### 5.2 VLAN配置（**重点命令**）
- **创建VLAN**：
  ```
  Switch(config)# vlan 10,20
  Switch(config-vlan)# exit
  Switch(config)# vlan 30
  Switch(config-vlan)# name caiwu    // 命名VLAN
  ```

- **将端口加入VLAN**：
  ```
  Switch(config)# interface f0/5
  Switch(config-if)# switchport mode access      // 定义二层端口模式
  Switch(config-if)# switchport access vlan 10   // 加入VLAN 10
  ```

- **批量将端口加入VLAN**：
  ```
  Switch(config)# interface range f0/1 - 10
  Switch(config-if-range)# switchport access vlan 10
  ```

- **还原接口默认配置**：
  ```
  Switch(config)# default interface f0/1
  ```

- **查看VLAN配置**：
  ```
  Switch# show vlan brief        // 查看所有VLAN摘要
  Switch# show vlan id vlan-id   // 查看指定VLAN信息
  ```

#### 5.3 Trunk原理与配置
- **Trunk的作用**：实现交换机之间的VLAN通信，只使用一条链路，通过标识区分不同VLAN数据

- **链路类型**：
  - **接入链路（Access）**：连接终端设备，只属于一个VLAN
  - **中继链路（Trunk）**：连接交换机，承载多个VLAN流量

- **VLAN标识方法**：
  - **ISL（Cisco私有标准）**：在帧前后添加26字节头和4字节尾
  - **IEEE 802.1q（标准）**：在帧中插入4字节Tag（**重点**，最常用）
    - TPID、Priority、CFI、VLAN ID（12位，可标识4096个VLAN）

- **Native VLAN**：
  - 不支持VLAN的交换机混合部署时使用
  - 允许转发未被标记的帧
  - **Cisco默认Native VLAN是VLAN 1**
  - **Trunk端口互联时，两端Native VLAN必须相同**

- **Trunk配置**：
  ```
  Switch(config-if)# switchport mode trunk
  Switch(config-if)# switchport trunk encapsulation dot1q  // 封装类型
  ```

#### 5.4 以太网通道（EtherChannel）
- 将多条物理链路捆绑为一条逻辑链路，增加带宽和冗余

---

## 6. 三层交换转发原理及VLAN间路由

### 内容概述
本PPT讲解三层交换机的转发原理，以及如何通过三层交换机实现VLAN间路由。

### 详细内容

#### 6.1 三层交换转发原理
- **三层交换机**：兼具二层交换和三层路由功能的设备
- **虚接口（SVI，Switched Virtual Interface）**：
  - 在三层交换机上配置的VLAN接口为虚接口
  - 每个VLAN对应一个虚接口
  - 虚接口的引入使得应用更加灵活

- **三层交换机VLAN间通信转发过程**：
  1. 主机A（VLAN 10）发送数据给主机B（VLAN 20）
  2. 数据到达三层交换机
  3. 三层交换机通过虚接口Int vlan 10接收数据
  4. 查询路由表，通过虚接口Int vlan 20转发
  5. 数据到达主机B

#### 6.2 三层交换机配置（**重点**）
- **启用路由功能**：
  ```
  Switch(config)# ip routing
  ```

- **配置虚接口IP地址**：
  ```
  Switch(config)# interface vlan 1
  Switch(config-if)# ip address 192.168.1.1 255.255.255.0
  Switch(config-if)# no shutdown
  ```

- **配置路由接口**（将二层端口改为三层端口）：
  ```
  Switch(config-if)# no switchport
  Switch(config-if)# ip address 10.1.1.1 255.255.255.252
  ```

- **查看路由表**：
  ```
  Switch# show ip route
  ```

#### 6.3 VLAN间路由实现
- **实现思路**：
  1. 在二层交换机上配置VLAN、Trunk
  2. 在三层交换机上配置VLAN、Trunk、虚接口IP
  3. 启用路由功能

- **配置示例**：
  ```
  // 三层交换机配置Trunk
  SW-3L(config)# interface fastEthernet 0/24
  SW-3L(config-if)# switchport trunk encapsulation dot1q
  SW-3L(config-if)# switchport mode trunk
  
  // 三层交换机配置虚接口
  SW-3L(config)# ip routing
  SW-3L(config)# interface vlan 1
  SW-3L(config-if)# ip address 192.168.1.1 255.255.255.0
  SW-3L(config-if)# no shut
  SW-3L(config)# interface vlan 2
  SW-3L(config-if)# ip address 192.168.2.1 255.255.255.0
  SW-3L(config-if)# no shut
  ```

#### 6.4 三层交换机配置路由
- **配置路由接口连接路由器**：
  ```
  SW-3L(config)# int f0/23
  SW-3L(config-if)# no switchport
  SW-3L(config-if)# ip address 10.1.1.1 255.255.255.252
  SW-3L(config)# ip route 0.0.0.0 0.0.0.0 10.1.1.2    // 默认路由
  ```

#### 6.5 DHCP中继配置
- **DHCP中继**：当DHCP服务器和客户端不在同一VLAN时，需要配置DHCP中继
  ```
  Switch(config-if)# ip helper-address DHCPsrv-IPAddress
  ```

---

## 7. STP工作原理及配置PVST+

### 内容概述
本PPT讲解生成树协议（STP）的工作原理、算法、收敛过程，以及PVST+的配置。

### 详细内容

#### 7.1 STP概述
- **STP（Spanning Tree Protocol，生成树协议）**：
  - 作用：防止交换网络中的环路
  - 环路危害：广播风暴、MAC地址表不稳定、多帧复制

- **BPDU（Bridge Protocol Data Unit，桥协议数据单元）**：
  - STP使用BPDU交换信息
  - 使用组播发送

#### 7.2 生成树算法（**重点**）
生成树选举三个步骤：

1. **选举根网桥（Root Bridge）**：
   - 比较网桥ID（Bridge ID = 优先级 + MAC地址）
   - **优先级取值范围0~65535，默认32768，步长4096**
   - 优先级小的成为根网桥；优先级相同则MAC地址小的成为根网桥

2. **选举根端口（Root Port，RP）**：
   - 在每个非根网桥上选举一个根端口
   - 选择标准：**到根网桥路径成本最小的端口**

3. **选举指定端口（Designated Port，DP）**：
   - 在每个网段上选择一个指定端口
   - 根网桥上的端口都是指定端口
   - 非根桥上的指定端口选择顺序：
     1. 根路径成本较低
     2. 所在交换机的网桥ID较小
     3. 端口ID较小（端口优先级+端口编号，优先级默认128）

- **阻塞端口**：既不是根端口也不是指定端口的端口被阻塞（Block）

#### 7.3 路径成本
| 链路带宽 | 路径成本（新） |
|---------|--------------|
| 10 Mbps | 100 |
| 100 Mbps | 19 |
| 1 Gbps | 4 |
| 10 Gbps | 2 |

#### 7.4 STP收敛
- **端口状态**：
  - Disabled：禁用
  - Blocking：阻塞，不转发数据，接收BPDU
  - Listening：监听，不转发数据，接收和发送BPDU
  - Learning：学习，不转发数据，学习MAC地址
  - Forwarding：转发，正常转发数据

- **收敛时间**：默认30~50秒（Forward Delay 15秒 × 2）

#### 7.5 PVST+配置
- **PVST+（Per-VLAN Spanning Tree Plus）**：每个VLAN一个生成树实例

- **配置命令**：
  ```
  Switch(config)# spanning-tree vlan vlan-id priority priority-value
  // 或
  Switch(config)# spanning-tree vlan vlan-id root {primary | secondary}
  ```

- **查看生成树状态**：
  ```
  Switch# show spanning-tree
  Switch# show spanning-tree vlan vlan-id
  ```

---

## 8. 静态路由原理

### 内容概述
本PPT讲解路由的基本原理、路由表的形成、静态路由和默认路由的配置，以及故障排查方法。

### 详细内容

#### 8.1 路由概述
- **路由**：跨越从源主机到目标主机的互联网络来转发数据包的过程
- **路由器工作原理**：根据路由表选择最佳路径转发数据包

#### 8.2 路由表的形成
- **直连路由（C）**：路由器接口配置IP并up后自动产生
- **静态路由（S）**：管理员手动配置
- **动态路由**：通过路由协议自动学习（RIP、OSPF等）

#### 8.3 静态路由配置（**重点**）
- **静态路由配置命令**：
  ```
  Router(config)# ip route network mask {address | interface}
  ```

- **配置实例一**（两台路由器互通）：
  ```
  R1(config)# ip route 30.0.0.0 255.255.255.0 20.0.0.2
  R2(config)# ip route 10.0.0.0 255.255.255.0 20.0.0.1
  ```

- **默认路由配置**（**重点**）：
  ```
  Router(config)# ip route 0.0.0.0 0.0.0.0 address
  ```
  - 0.0.0.0 0.0.0.0 代表任何网络
  - 常用于连接Internet的边界路由器

- **配置实例二**（内网访问Internet）：
  ```
  R1(config)# ip route 0.0.0.0 0.0.0.0 200.0.0.2    // R1配置默认路由
  R2(config)# ip route 192.168.1.0 255.255.255.0 200.0.0.1  // R2配置回程路由
  ```

#### 8.4 查看路由表
```
Router# show ip route
```
- **C**：直连路由
- **S**：静态路由
- **S***：默认路由

#### 8.5 故障排查（**重点**）
- **分层检查**：
  - 从物理层检查：查看端口状态排除接口、线缆问题
  - 查看IP地址和路由配置是否正确

- **分段检查**：
  - 将网络划分成多个小的段，逐段排除错误

- **常见故障**：
  - 物理层故障：线缆、接口问题
  - IP地址配置错误
  - 路由配置错误（缺少回程路由）

#### 8.6 实验
- **实验一**：配置静态路由实现全网互通
  - 非直连网段需要配置静态路由
  - 使用ping命令测试

- **实验二**：实现路由选路
  - 通过配置不同的静态路由实现负载均衡或主备路径

---

## 9. HSRP原理、配置和排错

### 内容概述
本PPT讲解热备份路由选择协议（HSRP）的工作原理、术语、配置方法和故障排查。

### 详细内容

#### 9.1 HSRP概述
- **HSRP（Hot Standby Router Protocol，热备份路由选择协议）**：
  - Cisco私有协议
  - 作用：实现网关冗余，提高网络可靠性
  - 原理：多台路由器组成虚拟路由器，共享虚拟IP和虚拟MAC

#### 9.2 HSRP相关概念（**重点**）
- **活跃路由器（Active Router）**：实际转发数据的路由器
- **备份路由器（Standby Router）**：监听活跃路由器状态，准备接管
- **虚拟路由器**：对外呈现的逻辑路由器，有虚拟IP和虚拟MAC
- **HSRP组**：一组参与HSRP的路由器，组号范围0~255

- **HSRP状态**：
  - Initial：初始状态
  - Learn：学习虚拟IP
  - Listen：监听活跃和备份路由器
  - Speak：参与选举
  - Standby：成为备份路由器
  - Active：成为活跃路由器

#### 9.3 HSRP配置（**重点命令**）
- **配置HSRP成员**：
  ```
  Switch(config-if)# standby group-number ip virtual-ip-address
  ```

- **配置优先级**（**重点**，默认100，范围0~255）：
  ```
  Switch(config-if)# standby group-number priority priority-value
  ```

- **配置占先权**（**重点**）：
  ```
  Switch(config-if)# standby group-number preempt
  ```
  - 优先级高的路由器重新获得转发权，恢复成为活跃路由器

- **配置计时器**：
  ```
  Switch(config-if)# standby group-number times hellotime holdtime
  ```
  - Hello时间默认3秒，保持时间默认10秒

- **配置端口跟踪**（**重点**）：
  ```
  Switch(config-if)# standby group-number track type mod/num interface-priority
  ```
  - 跟踪端口不可用时，HSRP优先级降低
  - 活跃路由器可以根据线路情况自动调整

#### 9.4 HSRP配置案例
```
// SW1配置（优先级200，成为活跃路由器）
SW1(config)# interface vlan 2
SW1(config-if)# ip address 192.168.1.1 255.255.255.0
SW1(config-if)# standby 10 ip 192.168.1.254
SW1(config-if)# standby 10 priority 200
SW1(config-if)# standby 10 preempt
SW1(config-if)# standby 10 timers 2 8
SW1(config-if)# standby 10 track fastEthernet 0/1 100

// SW2配置（优先级150，成为备份路由器）
SW2(config)# interface vlan 2
SW2(config-if)# ip address 192.168.1.2 255.255.255.0
SW2(config-if)# standby 10 ip 192.168.1.254
SW2(config-if)# standby 10 priority 150
SW2(config-if)# standby 10 preempt
SW2(config-if)# standby 10 timers 2 8
```

#### 9.5 查看HSRP状态
```
Switch# show standby brief          // 查看HSRP摘要信息
Switch# show standby                // 查看HSRP详细信息
```

#### 9.6 HSRP故障排查（**重点**）
- **常见故障**：
  1. HSRP组中交换机都处于初始状态：检查HSRP配置是否正确
  2. HSRP组中交换机都处于活跃状态：可能是VLAN隔离或组号不匹配
  3. 活跃设备上行链路故障，备份设备没有成为活跃状态：检查占先权配置
  4. 优先级高的路由器接入，没有成为活跃设备：检查占先权配置

- **HSRP与VRRP的区别**：
  - HSRP是Cisco私有协议，VRRP是标准协议
  - HSRP虚拟MAC：0000.0c07.acXX，VRRP虚拟MAC：0000.5e00.01XX

---

## 10. TCP、UDP 和ACL原理

### 内容概述
本PPT讲解TCP和UDP协议的报文格式、TCP连接建立和终止过程，以及ACL（访问控制列表）的基本原理和配置。

### 详细内容

#### 10.1 TCP协议（**重点**）
- **TCP（Transmission Control Protocol，传输控制协议）**：
  - 面向连接、可靠的传输协议
  - 提供流量控制、差错控制、拥塞控制

- **TCP报文首部格式**：
  - 源端口号（16位）、目标端口号（16位）
  - 序列号（32位）、确认号（32位）
  - 首部长度、标志位（URG、ACK、PSH、RST、SYN、FIN）
  - 窗口大小、校验和、紧急指针
  - 选项、数据

- **TCP三次握手建立连接**（**重点**）：
  1. 客户端发送SYN包（SYN=1，seq=x）
  2. 服务器回复SYN+ACK包（SYN=1，ACK=1，seq=y，ack=x+1）
  3. 客户端发送ACK包（ACK=1，seq=x+1，ack=y+1）

- **TCP四次握手终止连接**（**重点**）：
  1. 客户端发送FIN包（FIN=1，seq=u）
  2. 服务器回复ACK包（ACK=1，ack=u+1）
  3. 服务器发送FIN包（FIN=1，seq=v）
  4. 客户端回复ACK包（ACK=1，ack=v+1）
  - **为什么是四次？** 因为TCP连接是全双工的，每个方向都需要单独关闭

- **常用TCP端口号**：
  - 21：FTP
  - 22：SSH
  - 23：Telnet
  - 25：SMTP
  - 53：DNS
  - 80：HTTP
  - 443：HTTPS

#### 10.2 UDP协议
- **UDP（User Datagram Protocol，用户数据报协议）**：
  - 无连接、不可靠的传输协议
  - 花费开销小，传输效率高

- **UDP报文首部格式**：
  - 源端口号（16位）、目标端口号（16位）
  - UDP长度（16位）、UDP校验和（16位）

- **常用UDP端口号**：
  - 69：TFTP
  - 111：RPC
  - 123：NTP
  - 53：DNS

- **TCP与UDP区别**（**重点**）：
  | 特性 | TCP | UDP |
  |-----|-----|-----|
  | 连接 | 面向连接 | 无连接 |
  | 可靠性 | 可靠 | 不可靠 |
  | 传输效率 | 较低 | 较高 |
  | 应用场景 | 文件传输、网页浏览 | 视频流、语音通话 |

#### 10.3 ACL（访问控制列表）
- **ACL（Access Control List，访问控制列表）**：
  - 读取第三层、第四层包头信息
  - 根据预先定义的规则对包进行过滤

- **ACL工作原理**：
  - 数据包到达接口，按ACL规则逐条匹配
  - 匹配成功则执行允许或拒绝
  - 不匹配则继续匹配下一条
  - **最后隐含拒绝所有**（deny any）

- **ACL在接口应用的方向**：
  - **入方向（in）**：已到达路由器接口的数据包，将被路由器处理
  - **出方向（out）**：已经过路由器处理，正离开路由器接口的数据包

- **ACL类型**：
  - **标准ACL（1~99）**：基于源IP地址过滤
  - **扩展ACL（100~199）**：基于源IP、目的IP、协议、端口和标志过滤
  - **命名ACL**：使用名称代替表号

- **标准ACL配置**（**重点**）：
  ```
  // 创建ACL
  Router(config)# access-list 1 deny host 192.168.2.2
  Router(config)# access-list 1 permit any
  
  // 应用于接口
  Router(config-if)# ip access-group 1 in
  
  // 查看ACL
  Router# show access-lists
  ```

- **配置实例**：禁止主机PC2访问PC1，允许其他所有流量
  ```
  R1(config)# access-list 1 deny host 192.168.2.2
  R1(config)# access-list 1 permit any
  R1(config)# int f0/1
  R1(config-if)# ip access-group 1 in
  ```

- **关键字**：
  - `host`：表示单个主机
  - `any`：表示任何网络

---

## 11. NAT原理及配置

### 内容概述
本PPT讲解网络地址转换（NAT）的概念、实现方式、工作过程，以及静态NAT、动态NAT和PAT的配置和故障处理。

### 详细内容

#### 11.1 NAT概述
- **NAT（Network Address Translation，网络地址转换）**：
  - 作用：将私有IP地址转换为公有IP地址，实现内网访问Internet
  - 背景：IPv4地址不足，企业使用私有地址，需要访问Internet时进行转换

- **NAT实现方式**：
  - **静态转换（Static NAT）**：一对一固定映射
  - **动态转换（Dynamic NAT）**：从地址池中动态分配
  - **端口多路复用（PAT/NAT Overload）**：多对一，通过端口号区分

#### 11.2 NAT术语（**重点**）
- **内部局部IP地址（Inside Local）**：内网主机实际使用的私有IP
- **内部全局IP地址（Inside Global）**：内网主机对外呈现的公有IP
- **外部全局IP地址（Outside Global）**：外网主机的公有IP
- **外部局部IP地址（Outside Local）**：外网主机对内网呈现的IP

#### 11.3 静态NAT配置（**重点**）
- **应用场景**：内网服务器需要被外网访问（如Web服务器、邮件服务器）

- **配置步骤**：
  ```
  // 1. 设置外部端口IP
  Router(config)# interface FastEthernet 0/0
  Router(config-if)# ip address 61.159.62.130 255.255.255.248
  Router(config-if)# no shut
  
  // 2. 设置内部端口IP
  Router(config)# interface FastEthernet 1/0
  Router(config-if)# ip address 192.168.100.1 255.255.255.0
  Router(config-if)# no shut
  
  // 3. 建立静态地址转换
  Router(config)# ip nat inside source static 192.168.100.2 61.159.62.130
  Router(config)# ip nat inside source static 192.168.100.3 61.159.62.131
  
  // 4. 在端口上启用NAT
  Router(config)# interface FastEthernet 0/0
  Router(config-if)# ip nat outside
  Router(config)# interface FastEthernet 1/0
  Router(config-if)# ip nat inside
  
  // 5. 配置默认路由
  Router(config)# ip route 0.0.0.0 0.0.0.0 61.159.62.129
  ```

#### 11.4 NAT端口映射（**重点**）
- 将内部服务器的特定端口映射到外部IP的特定端口
  ```
  Router(config)# ip nat inside source static tcp 192.168.100.2 80 61.159.62.131 8080 extendable
  ```

#### 11.5 动态NAT配置
- **配置步骤**：
  ```
  // 1. 定义访问控制列表
  Router(config)# access-list 1 permit 192.168.100.0 0.0.0.255
  
  // 2. 定义合法IP地址池
  Router(config)# ip nat pool test0 61.159.62.131 61.159.62.190 netmask 255.255.255.192
  
  // 3. 实现网络地址转换
  Router(config)# ip nat inside source list 1 pool test0
  ```

#### 11.6 PAT配置（**重点**，最常用）
- **PAT（Port Address Translation）**：多对一地址转换，通过端口号区分不同会话

- **配置方式一**（使用地址池）：
  ```
  Router(config)# access-list 1 permit 10.1.1.0 0.0.0.255
  Router(config)# ip nat pool onlyone 61.159.62.131 61.159.62.131 netmask 255.255.255.248
  Router(config)# ip nat inside source list 1 pool onlyone overload   // overload启用端口复用
  ```

- **配置方式二**（复用外部接口地址，**最常用**）：
  ```
  Router(config)# access-list 1 permit 10.1.1.0 0.0.0.255
  Router(config)# ip nat inside source list 1 interface FastEthernet 0/0 overload
  ```

#### 11.7 验证NAT配置
```
Router# show ip nat translations [verbose]     // 查看NAT转换条目
Router# show ip nat statistics                 // 查看NAT统计信息
```

- **NAT转换条目超时时间**：
  - UDP：5分钟
  - DNS：1分钟
  - TCP：24小时

- **更改超时时间**：
  ```
  Router(config)# ip nat translation {dns-timeout | icmp-timeout | tcp-timeout | udp-timeout} {seconds | never}
  ```

- **清除NAT转换表**：
  ```
  Router# clear ip nat translation *             // 清除所有条目
  Router# clear ip nat translation inside local-ip global-ip
  ```
  - **静态NAT条目不会被清除**

#### 11.8 NAT故障处理（**重点**）
- **常见故障**：
  1. ACL阻止转换后的流量
  2. 进行地址转换的ACL不全
  3. **overload参数漏配**（导致PAT无法正常工作）
  4. 不对称路由问题
  5. 动态地址池IP地址范围配置错误
  6. 动态地址池与静态转换地址重叠
  7. **Inside和outside接口配置错误**

- **故障排除步骤**：
  1. 检查物理设备和NAT配置
  2. 通过show命令查看NAT的各种信息
  3. 通过`debug ip nat`命令跟踪NAT操作

---

## 12. VPN原理及配置

### 内容概述
本PPT讲解VPN（虚拟专用网）的定义、模式与类型、加密算法、数据报文验证，以及IPSec VPN的原理和配置。

### 详细内容

#### 12.1 VPN概述
- **VPN（Virtual Private Network，虚拟专用网）**：
  - 定义：在公共网络（如Internet）上建立安全的私有通信通道
  - 作用：实现机密信息在Internet上安全传输

- **VPN连接模式**：
  - **传输模式**：只加密IP数据包的数据部分，不加密IP头部
  - **隧道模式**：加密整个IP数据包，并添加新的IP头部

- **VPN类型**：
  - **站点到站点VPN（Site-to-Site）**：连接两个网络（如分公司和总公司）
  - **远程访问VPN（Remote Access）**：单个用户远程接入企业网络

#### 12.2 加密算法（**重点**）
- **对称加密算法**：加密和解密使用相同密钥
  - **DES**：数据加密标准，56位密钥（已不安全）
  - **3DES**：三重DES，168位密钥
  - **AES**：高级加密标准，128/192/256位密钥（**推荐使用**）

- **非对称加密算法**：使用公钥和私钥对
  - **DH算法（Diffie-Hellman）**：密钥交换算法
  - 公钥加密，私钥解密
  - **私钥始终未在网上传输**

- **加密算法的应用**（**重点**）：
  - 问题：对称加密密钥可能被窃听；非对称加密计算复杂、效率低
  - **解决方案**：通过非对称加密算法加密对称加密算法的密钥，再用对称加密算法加密实际数据

#### 12.3 数据报文验证
- **HMAC（Hash-based Message Authentication Code）**：
  - 实现数据完整性验证
  - 实现身份验证

- **Hash算法**：
  - **MD5**：128位摘要
  - **SHA**：160位摘要（更安全）

- **数字签名**：
  - 发送方用私钥对数据Hash值加密
  - 接收方用公钥解密，与本地Hash值比对
  - 如果数据被篡改将无法得到相同的数字签名

#### 12.4 IPSec VPN原理（**重点**）
- **IPSec（IP Security）**：为IP通信提供安全性的协议族

- **建立IPSec VPN连接需要3个步骤**：
  1. 流量触发IPSec
  2. 建立管理连接（ISAKMP/IKE阶段1）
  3. 建立数据连接（ISAKMP/IKE阶段2）

#### 12.5 ISAKMP/IKE阶段1（**重点**）
- **三个任务**：
  1. 协商采用何种方式建立管理连接（传输集）
  2. 通过DH算法共享密钥信息
  3. 对等体彼此进行身份验证

- **传输集包含**：
  1. 加密算法
  2. HMAC功能
  3. 设备验证的类型
  4. DH密钥组
  5. 管理连接的生存周期

- **身份验证方式**：
  - 预共享密钥（Pre-Shared Key）
  - RSA数字签名

#### 12.6 ISAKMP/IKE阶段1配置
```
Router(config)# crypto isakmp policy priority
Router(config-isakmp)# encryption {des | 3des | aes}      // 指定加密算法
Router(config-isakmp)# hash {sha | md5}                   // 指定hash算法
Router(config-isakmp)# authentication pre-share          // 采用预共享密钥
Router(config-isakmp)# group {1 | 2 | 5}                 // 指定DH算法密钥长度
Router(config-isakmp)# lifetime seconds                    // 生存周期
```

- **查看ISAKMP策略**：
  ```
  Router# show crypto isakmp policy
  ```

- **配置预共享密钥**：
  ```
  Router(config)# crypto isakmp key {0 | 6} keystring address peer-address [subnet_mask]
  ```

#### 12.7 ISAKMP/IKE阶段2
- **需要完成的任务**：
  1. 定义对等体间需要保护何种流量
  2. 定义用来保护数据的安全协议
  3. 定义传输模式（传输模式/隧道模式）
  4. 定义数据连接的生存周期以及密钥刷新方式

- **安全协议**：
  - **AH（Authentication Header）**：提供数据完整性验证和身份验证，不加密
  - **ESP（Encapsulating Security Payload）**：提供加密、数据完整性验证和身份验证

#### 12.8 IPSec VPN配置实现
- **配置步骤概述**：
  1. 配置ISAKMP/IKE阶段1（管理连接）
  2. 配置ISAKMP/IKE阶段2（数据连接）
  3. 配置加密映射（Crypto Map）
  4. 将加密映射应用到接口

- **查看IPSec配置**：
  ```
  Router# show crypto isakmp sa      // 查看ISAKMP安全关联
  Router# show crypto ipsec sa       // 查看IPSec安全关联
  ```

---

## 附录：常用命令速查表

### 基础配置命令
| 命令 | 作用 |
|-----|------|
| `en` | 进入特权模式 |
| `conf t` | 进入全局配置模式 |
| `no ip domain-lookup` | 禁用DNS查询 |
| `line con 0` | 进入控制台线路配置 |
| `logging synchronous` | 同步日志输出 |
| `exec-timeout 0 0` | 禁用空闲超时 |
| `service password-encryption` | 加密明文密码 |

### 接口配置命令
| 命令 | 作用 |
|-----|------|
| `interface type slot/port` | 进入接口配置 |
| `ip address ip mask` | 配置IP地址 |
| `no shutdown` | 开启接口 |
| `no switchport` | 将二层口改为三层口 |
| `switchport mode access` | 设置接入模式 |
| `switchport access vlan id` | 加入VLAN |
| `switchport mode trunk` | 设置Trunk模式 |
| `switchport trunk encapsulation dot1q` | 设置Trunk封装 |

### VLAN与路由命令
| 命令 | 作用 |
|-----|------|
| `vlan id` | 创建VLAN |
| `interface vlan id` | 进入VLAN虚接口 |
| `ip routing` | 启用三层路由 |
| `show vlan brief` | 查看VLAN信息 |
| `show ip route` | 查看路由表 |
| `ip route net mask next-hop` | 配置静态路由 |
| `ip route 0.0.0.0 0.0.0.0 next-hop` | 配置默认路由 |

### NAT命令
| 命令 | 作用 |
|-----|------|
| `ip nat inside source static local global` | 静态NAT |
| `ip nat inside source list acl pool name` | 动态NAT |
| `ip nat inside source list acl interface overload` | PAT |
| `ip nat inside/outside` | 启用NAT方向 |
| `show ip nat translations` | 查看NAT转换 |

### HSRP命令
| 命令 | 作用 |
|-----|------|
| `standby group ip vip` | 配置HSRP组 |
| `standby group priority val` | 配置优先级 |
| `standby group preempt` | 配置占先权 |
| `standby group track interface val` | 配置端口跟踪 |
| `show standby brief` | 查看HSRP状态 |

### 安全命令
| 命令 | 作用 |
|-----|------|
| `access-list num permit/deny source` | 配置标准ACL |
| `ip access-group num in/out` | 应用ACL |
| `crypto isakmp policy num` | 配置ISAKMP策略 |
| `crypto isakmp key key address peer` | 配置预共享密钥 |
| `show crypto isakmp policy` | 查看ISAKMP策略 |

---

> **文档说明**：本文档基于11个网络技术培训PPT整理，涵盖了从网络基础到高级配置的完整知识体系。建议结合实际设备进行实验操作，加深理解。
