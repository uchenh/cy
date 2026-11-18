# Rocky_Linux_9.5_安装教程

安装 Rocky Linux 9.5 系统

图文完整安装教程

基于 VMware Workstation 17.6.1 Pro 的虚拟机安装实践

共 40 张操作截图，全文图文对照

整理日期：2026 年 9 月


## 目录


## 文档说明

本文档整理自网络公开的《安装 RockyLinux 9.5 系统》图文教程，完整保留了原文的安装步骤与全部操作截图，并按照“准备 → 建虚拟机 → 安装 → 检查”的顺序重新组织章节结构，便于离线查阅与打印使用。

Rocky Linux 是 CentOS 停止维护后主流的替代发行版之一，由 CentOS 创始人主导开发，与 RHEL（Red Hat Enterprise Linux）保持二进制兼容，可视为 CentOS 的平滑接续方案。本文以在 VMware Workstation 中安装 Rocky Linux 9.5 为例，逐步说明从下载镜像到系统安装完成的完整过程。

说明：文中出现的所有截图均对应原文中的原始配图，按原文顺序插入到相应步骤之后；图号（图 1、图 2 …）为便于查阅而新增的编号，原文未提供图注。


### Rocky9.5安装步骤


## 1、准备工作


#### 1. 准备好虚拟机

虚拟机：VMware Workstation 17.6.1 Pro


#### 2. 下载Rocky9.5版本

获取Rocky路径

rockylinux.org/download

获取Rocky路径

rockylinux.org/download

https://download.rockylinux.org/pub/rocky/9/isos/x86_64/Rocky-9.5-x86_64-dvd.iso

# ISO各个版本的区别

XXX-X-x86_64-DVD.iso 标准安装版

XXX-X-x86_64-Minimal.iso 精简版，也可以说是最小化安装，自带的软件最少

图 1


## 2、创建新的虚拟机

图 2

图 3

图 4

图 5

图 6

图 7

图 8

图 9

桥接网络：将虚拟机的网络接口直接连接到物理网络。虚拟机会像一台真正的物理主机一样，与宿主机（Host）处于同一个网络中。并且局域网其他主机能够访问连接到此主机。

NAT网络：将虚拟机的网络流量通过宿主机进行转发，使虚拟机可以访问外部网络，但外部网络无法直接访问虚拟机。

桥接网络：将虚拟机的网络接口直接连接到物理网络。虚拟机会像一台真正的物理主机一样，与宿主机（Host）处于同一个网络中。并且局域网其他主机能够访问连接到此主机。

NAT网络：将虚拟机的网络流量通过宿主机进行转发，使虚拟机可以访问外部网络，但外部网络无法直接访问虚拟机。

图 10

图 11

图 12

图 13

图 14

图 15

图 16

图 17

图 18


## 3、开始安装


#### 1· 启动、选择安装项

# 1· 选择第一个选项

# 2· 网卡名称变eth0

# 在输入quit后面输入(quit与net有空格)

net.ifnames=0 biosdevname=0

图 19

图 20


#### 2· 选择安装语言

然后单击继续（continue)

图 21


#### 3· 存储配置

为存储配置（storage configuration)选择自动（automatic）或自定义（custom)选项

自动（automatic）:安装程序将在磁盘上自动创建分区

自定义（custom):允许我们在磁盘上手动创建分区

1·点击自定义（Custom）选项

2·点击执行（Done）

3·选择标准分区方案（Standard Partition）

4·点击“+”按钮添加分区信息，然后点击添加挂载点（Add mount point）

创建以下分区

/boot: 1GB

/ :剩下所有的

图 22

图 23

图 24

图 25

选择接受更改（Accept Changes）按钮将更改写入磁盘，同时返回安装界面

图 26


#### 4· 用户配置

设置root 密码（Root Password），点击执行（Done)

图 27

图 28


#### 5· 环境配置

点击软件选择（Software Selection)按钮

图 29

选择基本环境（Base Environment）：

图 30


#### 6· 网络配置

选择网络和主机名（Network & Host Name)，开启网卡，设置主机名:rocky-00，点击应用（apply），点击执行（Done)

图 31

图 32


#### 7· 时区配置

设置时区（Time & Date)，点击地图上的上海，点击执行（Done)

图 33

图 34


#### 8. 关闭KDUMP

图 35

图 36


#### 9· 开始安装

准备开始安装：点击开始安装（Begin installation)按钮

图 37

安装程序已经开始安装

图 38

安装完成，点击重新启动系统（Reboot System)

图 39


## 4、系统检查

图 40


## 附：安装要点回顾

虚拟机软件：VMware Workstation 17.6.1 Pro。

镜像获取：rockylinux.org/download 或国内镜像站（如阿里云镜像），选择 DVD 标准安装版。

启动参数：在安装引导界面输入 net.ifnames=0 biosdevname=0，使网卡名称规范为 eth0。

分区方案：/boot 分配 1GB，根分区 / 使用剩余全部空间。

网络配置：开启网卡并设置主机名（示例为 rocky-00）。

时区设置：选择上海（Asia/Shanghai）。

安全建议：设置强 root 密码，如无特殊需求可关闭 KDUMP 以节省内存。

安装完成后重启系统，即可进入 Rocky Linux 9.5 的登录界面。

本文档共插入 40 张教程截图。


> 本文档含 40 张插图，见 `assets/Rocky_Linux_9.5_安装教程/`。

![图 1](assets/Rocky_Linux_9.5_安装教程/img1.png)

![图 2](assets/Rocky_Linux_9.5_安装教程/img2.png)

![图 3](assets/Rocky_Linux_9.5_安装教程/img3.png)

![图 4](assets/Rocky_Linux_9.5_安装教程/img4.png)

![图 5](assets/Rocky_Linux_9.5_安装教程/img5.png)

![图 6](assets/Rocky_Linux_9.5_安装教程/img6.png)

![图 7](assets/Rocky_Linux_9.5_安装教程/img7.png)

![图 8](assets/Rocky_Linux_9.5_安装教程/img8.png)

![图 9](assets/Rocky_Linux_9.5_安装教程/img9.png)

![图 10](assets/Rocky_Linux_9.5_安装教程/img10.png)

![图 11](assets/Rocky_Linux_9.5_安装教程/img11.png)

![图 12](assets/Rocky_Linux_9.5_安装教程/img12.png)

![图 13](assets/Rocky_Linux_9.5_安装教程/img13.png)

![图 14](assets/Rocky_Linux_9.5_安装教程/img14.png)

![图 15](assets/Rocky_Linux_9.5_安装教程/img15.png)

![图 16](assets/Rocky_Linux_9.5_安装教程/img16.png)

![图 17](assets/Rocky_Linux_9.5_安装教程/img17.png)

![图 18](assets/Rocky_Linux_9.5_安装教程/img18.png)

![图 19](assets/Rocky_Linux_9.5_安装教程/img19.png)

![图 20](assets/Rocky_Linux_9.5_安装教程/img20.png)

![图 21](assets/Rocky_Linux_9.5_安装教程/img21.png)

![图 22](assets/Rocky_Linux_9.5_安装教程/img22.png)

![图 23](assets/Rocky_Linux_9.5_安装教程/img23.png)

![图 24](assets/Rocky_Linux_9.5_安装教程/img24.png)

![图 25](assets/Rocky_Linux_9.5_安装教程/img25.png)

![图 26](assets/Rocky_Linux_9.5_安装教程/img26.png)

![图 27](assets/Rocky_Linux_9.5_安装教程/img27.png)

![图 28](assets/Rocky_Linux_9.5_安装教程/img28.png)

![图 29](assets/Rocky_Linux_9.5_安装教程/img29.png)

![图 30](assets/Rocky_Linux_9.5_安装教程/img30.png)

![图 31](assets/Rocky_Linux_9.5_安装教程/img31.png)

![图 32](assets/Rocky_Linux_9.5_安装教程/img32.png)

![图 33](assets/Rocky_Linux_9.5_安装教程/img33.png)

![图 34](assets/Rocky_Linux_9.5_安装教程/img34.png)

![图 35](assets/Rocky_Linux_9.5_安装教程/img35.png)

![图 36](assets/Rocky_Linux_9.5_安装教程/img36.png)

![图 37](assets/Rocky_Linux_9.5_安装教程/img37.png)

![图 38](assets/Rocky_Linux_9.5_安装教程/img38.png)

![图 39](assets/Rocky_Linux_9.5_安装教程/img39.png)

![图 40](assets/Rocky_Linux_9.5_安装教程/img40.png)
