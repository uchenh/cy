# Lustre Ubuntu 24.04 部署指南（方法论与原理）

> 本文档提供 **Lustre 在 Ubuntu 24.04（noble，内核 6.8 GA）** 上的完整部署指南，覆盖：支持现状与版本选择、架构方案选择（方案 A：服务端用 RHEL 家族 + Ubuntu 客户端；方案 B：全栈 Ubuntu 源码编译）、客户端安装（4 条路径）、服务端源码编译（ZFS / ldiskfs 双路线）、后续部署流程差异与 Ubuntu 特有常见问题排查。文中命令默认在 Ubuntu 24.04 上执行；Lustre 版本统一为 **2.16.x / 2.17.x**，**不使用 2.15**（2.15 与 kernel 6.x 不兼容）。

---

## 1. 支持现状与版本选择（Ubuntu 24.04）

Ubuntu 24.04（代号 noble）是目前最活跃的 LTS 发行版之一，也是 Lustre 官方支持矩阵中最新的 Ubuntu 版本。Lustre 的代码与内核深度耦合，动手前必须先确认「发行版 + Lustre 版本 + 内核版本」三者匹配，否则编译内核模块或 `modprobe lustre` 时会有大量踩坑。

### 1.1 官方支持现状

以下是 Whamcloud/DDN（Lustre 官方）对 Ubuntu 24.04 的支持现状：

| 维度 | 支持现状 |
| --- | --- |
| 客户端 | 从 **Lustre 2.16** 起官方支持，属 **patchless 客户端**——无需补丁内核，直接基于发行版原装内核编译/加载模块即可 |
| 服务端 | 从 **Lustre 2.16** 起被列入官方支持矩阵；2.16 ChangeLog 将 Ubuntu 24.04 内核 `6.8.0-38`、`6.10.0-15` 列为 **server primary kernels** |
| 预编译包 | 官方**不提供** Ubuntu 的预编译服务端 `.deb`，服务端**必须源码编译**（见第 4 章） |
| 官方 apt 仓库 | 无官方通用 apt 仓库；客户端可从 Azure Managed Lustre / Google Cloud Artifact Registry / 第三方镜像获取（见 §3） |

> 参考：[Whamcloud Lustre Support Matrix](https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix) 中 2.16 / 2.17 的客户端行均包含 Ubuntu 24.04；各版本 ChangeLog 见 [wiki.lustre.org](https://wiki.lustre.org/)。

### 1.2 重要版本警告：Lustre 2.15 LTS 与 Linux kernel 6.x 不兼容

**Ubuntu 24.04 上绝对不要使用 Lustre 2.15。** 依据如下：

| 依据 | 说明 |
| --- | --- |
| Oracle 官方文档 | 明确声明 **Lustre 2.15.5 与 Linux kernel 6 不兼容**，客户端需要 5.15.x 内核（[Lustre Clients for Ubuntu](https://docs.oracle.com/en-us/iaas/Content/lustre/clients-for-ubuntu.htm)） |
| 2.15 LTS 支持矩阵 | 2.15.x 客户端最高只覆盖 **Ubuntu 22.04**，服务端最高只覆盖 RHEL 8.10 / 9.x，均未包含 Ubuntu 24.04 |
| 内核换代 | 2.15 LTS 从发布（2022-06）起就基于 5.x 内核设计，无法直接适配 6.x 内核 |

| Lustre 版本 | 类型 | Ubuntu 24.04（kernel 6.x）支持 | 结论 |
| --- | --- | --- | --- |
| 2.15.x | LTS | 不支持（官方明确 2.15.5 不兼容 kernel 6） | **不要用** |
| 2.16.x | feature 分支 | 支持（客户端 primary `6.8.0-35`；服务端 primary `6.8.0-38` / `6.10.0-15`） | 推荐 |
| 2.17.x | 最新 feature 分支（master） | 支持 | 推荐 |

### 1.3 推荐版本组合

| 角色 | 发行版 | 内核 | Lustre 版本 | 获取方式 |
| --- | --- | --- | --- | --- |
| 客户端（方案 A / B 通用） | Ubuntu 24.04（noble） | 6.8 GA 内核（`6.8.0-*-generic`） | 2.16.x / 2.17.x | DKMS 仓库或源码编译（§3） |
| 服务端（方案 B，全栈 Ubuntu） | Ubuntu 24.04（noble） | 6.8 GA 内核 | 2.16.x / 2.17.x | 源码编译（第 4 章） |
| 服务端（方案 A，RHEL 家族） | Rocky/RHEL 8.10 | Whamcloud 补丁内核 `4.18.0-*el8_lustre` | 2.15.8 | 官方预编译包（见 `docs/04-lustre-虚拟机部署指南.md`） |

> **互操作提示**：方案 A 中 Ubuntu 24.04 客户端装 2.16.x、Rocky 服务端为 2.15.8，属于官方支持的互操作范围——2.16 ChangeLog 的互操作性声明为「Clients & Servers: Latest 2.15.X」。

### 1.4 GA 内核 vs HWE 内核

Ubuntu 24.04 的镜像（尤其是 Azure 等云厂商镜像）默认安装的内核分两类，必须分清：

| 类型 | 代表包 | 支持周期 | 与 Lustre 的配合 |
| --- | --- | --- | --- |
| **GA / LTS 内核** | `linux-image-generic`（`6.8.0-*-generic`） | 随 24.04 LTS 长期维护 | 各客户端仓库（Azure/GCP/镜像）的模块均针对 6.8 线提供，**推荐** |
| **HWE 内核** | `linux-image-azure`（Azure 默认）、`linux-image-generic-hwe-24.04` | **每个 HWE 内核仅支持约 6 个月**，随后被更新的内核线替换 | Lustre 适配跟不上 HWE 的迭代节奏，HWE 一升级模块就编译失败或加载不上 |

> 以 Azure 为例：Azure Marketplace 的 Ubuntu 24.04 镜像默认使用 HWE（`linux-image-azure`）内核；官方文档明确建议切换到 LTS 内核（Azure 上为 `linux-image-azure-lts-24.04`），因为它保持在 6.8 这条 Lustre 已支持的内核线上。

**建议**：所有节点统一切换回 **GA/LTS 内核（6.8）**，并锁定内核版本，避免 `apt` 自动升级换内核导致 Lustre 模块失效：

```bash
# 1) 查看当前内核与已安装的内核包
uname -r
dpkg -l | grep -E 'linux-(image|headers)' | awk '{print $2}'

# 2) 安装 GA 内核与配套头文件（与 -generic 对齐）
sudo apt update
sudo apt install -y linux-image-generic linux-headers-generic

# 3) 移除默认的 HWE 内核元包（Azure 镜像为 linux-image-azure；其他平台按实际 HWE 包名调整）
sudo apt remove -y linux-image-azure linux-headers-azure

# 4) 重启进入 6.8.x-generic 内核
sudo reboot
uname -r    # 期望输出形如 6.8.0-XX-generic

# 5) 锁定内核，防止日后 apt upgrade 自动换内核（可选但强烈建议）
sudo apt-mark hold linux-image-generic linux-headers-generic
```

---

## 2. 架构方案选择

核心选型问题是：**是否让 Ubuntu 24.04 承担服务端（MGS/MDS/OSS）角色**。两条方案的客户端安装方法完全一致（§3），区别只在服务端如何获得。

### 2.1 方案 A：服务端用 RHEL 家族，Ubuntu 只做客户端（推荐入门）

| 项目 | 内容 |
| --- | --- |
| 服务端（MGS/MDS/OSS） | Rocky/RHEL 8.10 + Lustre 2.15.8 **官方预编译包**，详见工作区 `docs/04-lustre-虚拟机部署指南.md` |
| 客户端 | Ubuntu 24.04 + Lustre 2.16.x / 2.17.x（本部分 §3） |
| 互操作性 | 2.16 客户端 ↔ 2.15.8 服务端，官方 2.16 ChangeLog 声明覆盖 Latest 2.15.X |
| 主要代价 | 无（无需源码编译服务端，部署最快、踩坑最少） |
| 适用场景 | 入门学习、快速验证、生产评估前的原型环境 |

### 2.2 方案 B：全栈 Ubuntu（服务端源码编译）

| 项目 | 内容 |
| --- | --- |
| 全栈 | MGS/MDS/OSS/客户端全部 Ubuntu 24.04，系统与命令集统一 |
| 服务端获取 | 从源码编译（`git clone` → `autogen.sh` → `configure` → `make debs`），详细步骤见第 4 章 |
| 主要代价 | **编译服务端耗时约数小时**（单机通常 2~6 小时，视 CPU/内存而定）；要求内核头文件与运行内核**严格一致**；依赖多、首次失败率高；换内核后需重编 |
| 适用场景 | 统一 Ubuntu 技术栈、需要深入内核模块源码、对服务端形态有定制需求的团队 |

### 2.3 拓扑说明与推荐实验拓扑

- **单节点**：MGS+MDS+OSS+Client 全部合并到 1 台 VM，最快跑通「元数据 + 数据 + 挂载」全链路；
- **多节点（推荐）**：3 台 VM——`mds1`（MGS+MDS）、`oss1`（OSS）、`client1`（客户端），贴近生产形态，可验证 LNet 网络通信；
- **进阶**：4 台 VM——`mds1`、`oss1`、`oss2`、`client1`，用于验证多 OST 条带化与 OST 故障影响。

| 拓扑 | 组成 | 适合场景 |
| --- | --- | --- |
| 单节点（1 台 VM） | 所有角色合一 | 资源紧张、快速学习命令 |
| **推荐（3 台 VM）** | mds1（MGS+MDS）/ oss1（OSS）/ client1 | 模拟真实部署、验证网络通信与故障隔离 |
| 进阶（4 台 VM） | mds1 / oss1 / oss2 / client1 | 多 OST 条带化、OST 故障演练 |

### 2.4 方案对比小结

| 对比项 | 方案 A | 方案 B |
| --- | --- | --- |
| 部署时间 | 半天内可跑通 | 服务端编译数小时起步 |
| 服务端维护 | 官方预编译包，升级简单 | 源码编译，需自建构建流程 |
| 内核升级 | 锁定 Whamcloud 补丁内核 | 需重编内核模块，升级非常谨慎 |
| 技术栈统一性 | 混合（Rocky 服务端 + Ubuntu 客户端） | 全 Ubuntu 统一 |
| 一句话结论 | **入门首选** | 有明确技术栈/源码需求时选 |

---

## 3. Ubuntu 24.04 客户端安装（方案 A 与 B 通用）

客户端是 **patchless** 的，不需要补丁内核。Ubuntu 24.04 没有官方通用 apt 仓库，但有 4 条可用路径：Azure Managed Lustre 仓库、Google Cloud Artifact Registry、lustre.software 镜像、源码编译——**任选其一即可**（优先路径 1 或 2）。

### 3.1 安装前检查

```bash
# 1) 确认内核版本（应为 6.8.x-generic，GA 内核，见 §1.4）
uname -r

# 2) 确认内核头文件已安装且与运行内核严格一致（DKMS/编译的硬前提）
apt list --installed | grep linux-headers
# 期望看到形如 linux-headers-6.8.0-XX-generic 的包，版本号与 uname -r 完全一致；
# 若缺失，先安装：sudo apt install -y linux-headers-$(uname -r)

# 3) 确认编译工具链（DKMS 首次编译需要）
gcc --version
make --version

# 4) 确认 GLIBC 版本（lustre-client-utils 需要 GLIBC 2.38+）
ldd --version | head -1    # Ubuntu 24.04 自带 glibc 2.39，满足要求
```

| 检查项 | 命令 | 要求 |
| --- | --- | --- |
| 内核 | `uname -r` | `6.8.0-XX-generic`（GA 内核，不要 HWE） |
| 内核头文件 | `apt list --installed \| grep linux-headers` | 与运行内核**严格一致**，否则模块编译/加载失败 |
| 编译工具链 | `gcc --version` / `make --version` | 已安装 |
| GLIBC | `ldd --version` | ≥ 2.38（noble 为 2.39，满足） |

### 3.2 路径 1：Azure Managed Lustre 仓库（推荐，Lustre 2.17，amd64/arm64）

来源：Microsoft 官方，仓库地址 `https://packages.microsoft.com/repos/amlfs-<codename>/`（noble 即 `amlfs-noble`），支持 amd64 与 arm64。仓库同时提供两种包：

| 包形态 | 说明 | 适用内核 |
| --- | --- | --- |
| `amlfs-lustre-client-dkms-<ver>` | DKMS 包，安装时按当前内核自动编译，跨内核通用 | **任意内核（GA 下推荐）** |
| `amlfs-lustre-client-<ver>=<kernel>` | 预编译 kmod 元包，需指定精确内核版本 | 仅覆盖特定 `-azure` 内核 |

```bash
# 1) 配置软件源与密钥
sudo apt update
sudo apt install -y ca-certificates curl apt-transport-https lsb-release gnupg dpkg-dev
source /etc/lsb-release
ARCH=$(dpkg-architecture -q DEB_BUILD_ARCH)
echo "deb [arch=${ARCH}] https://packages.microsoft.com/repos/amlfs-${DISTRIB_CODENAME}/ ${DISTRIB_CODENAME} main" \
  | sudo tee /etc/apt/sources.list.d/amlfs.list
curl -sL https://packages.microsoft.com/keys/microsoft.asc \
  | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null
sudo apt update

# 2) 查看仓库中可用的 DKMS 包（包名带版本号后缀，版本以仓库实际为准）
apt list -a 'amlfs-lustre-client*' | grep dkms

# 3) 安装 DKMS 客户端（示例版本为撰写时的 2.17.0；请以第 2 步查询结果为准）
sudo apt install -y amlfs-lustre-client-dkms-2.17.0-24-gf517bc4
# DKMS 按当前内核自动编译，首次需几分钟（依赖 gcc / make / linux-headers-$(uname -r)）
```

> **注意**：预编译 kmod（`amlfs-lustre-client-<ver>=<kernel>`）只覆盖特定 `-azure` 内核；使用 GA `-generic` 内核时务必选 **DKMS 包**。仓库路径与包名以 packages.microsoft.com 官网最新为准。

### 3.3 路径 2：Google Cloud Artifact Registry（lustre-client-ubuntu-noble）

来源：Google Cloud 官方托管，DKMS 仓库项目 `lustre-client-modules-dkms`，Ubuntu 24.04 对应仓库 `lustre-client-ubuntu-noble`，提供 `lustre-client-modules-dkms`（DKMS 编译）+ `lustre-client-utils`（用户态工具）。

```bash
# 1) 安装 Artifact Registry 的 apt 传输插件
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/google-cloud.gpg
echo 'deb [signed-by=/usr/share/keyrings/google-cloud.gpg] http://packages.cloud.google.com/apt apt-transport-artifact-registry-stable main' \
  | sudo tee /etc/apt/sources.list.d/artifact-registry.list
sudo apt update && sudo apt install -y apt-transport-artifact-registry

# 2) 配置 noble 客户端仓库并更新索引
curl -fsSL https://us-apt.pkg.dev/doc/repo-signing-key.gpg \
  | sudo gpg --dearmor -o /usr/share/keyrings/lustre-client.gpg
echo "deb [signed-by=/usr/share/keyrings/lustre-client.gpg] ar+https://us-apt.pkg.dev/projects/lustre-client-modules-dkms lustre-client-ubuntu-noble main" \
  | sudo tee -a /etc/apt/sources.list.d/artifact-registry.list
sudo apt update

# 3) 安装客户端（DKMS 模块 + 用户态工具，首次编译需几分钟）
sudo apt install -y lustre-client-modules-dkms/lustre-client-ubuntu-noble
sudo apt install -y lustre-client-utils/lustre-client-ubuntu-noble
```

> **注意**：该仓库为 GCP Compute Engine 场景设计，`ar+https://` 传输可能依赖 GCP VM 的凭证/访问范围；非 GCP 机器上如 `apt update` 失败，请改用路径 1 或 4。仓库配置以 Google Cloud 官方文档最新为准。

### 3.4 路径 3：lustre.software 镜像（ubuntu2404 client，feature release，非官方）

来源：社区镜像（**非官方**），镜像 Whamcloud feature 分支构建；`ubuntu2404` 对应 **feature release**（2.16/2.17 开发线）。该镜像目录只提供**预编译**模块包（撰写时覆盖 `6.8.0-31-generic`、`6.8.0-35-generic`），**没有 DKMS 包**。

> **风险提示**：镜像未提供 GPG 密钥，必须关闭签名校验（apt 中即 `[trusted=yes]`），存在被篡改的风险；仅建议学习/实验环境使用。

```bash
# 1) 配置软件源（trusted=yes 关闭签名校验；路径以 lustre.software 镜像目录最新为准）
echo 'deb [trusted=yes] https://lustre.software/mirror/latest-feature-release/latest-feature-release/ubuntu2404/client/ ./' \
  | sudo tee /etc/apt/sources.list.d/lustre-feature.list
sudo apt update

# 2) 安装用户态工具 + 与运行内核匹配的预编译模块
sudo apt install -y lustre-client-utils
sudo apt install -y lustre-client-modules-$(uname -r)
# 若仓库没有与 uname -r 完全一致的模块包，请改用 DKMS 路径（§3.2/§3.3）或源码编译（§3.5）
```

### 3.5 路径 4：源码编译（备选）

适用场景：上述仓库均没有合适的包、需要特定分支/提交，或想研究客户端源码。客户端无需补丁内核，直接用发行版内核头文件即可。

```bash
# 1) 准备构建依赖
sudo apt install -y build-essential git autoconf automake libtool \
  linux-headers-$(uname -r) libreadline-dev libmount-dev libyaml-dev libnl-3-dev \
  libnl-genl-3-dev libkrb5-dev libkeyutils-dev libssl-dev libhwloc-dev \
  libsnmp-dev libpython3-dev flex bison pkg-config libaio-dev

# 2) 获取源码并检出目标分支（b2_16 = 2.16.x；master = 2.17.x/最新）
git clone https://github.com/lustre/lustre-release.git
cd lustre-release
git checkout b2_16          # 或 git checkout master

# 3) 生成 configure 并仅构建客户端（--with-linux 指向当前内核头文件）
sh autogen.sh
./configure --disable-server --with-linux=/usr/src/linux-headers-$(uname -r)

# 4) 打包为 .deb（两种方式任选：debs 绑定当前内核；dkms-debs 跨内核更省事）
make debs
# 或
make dkms-debs

# 5) 安装生成的 .deb 包
sudo apt install -y ./debs/*.deb

# 6) 重建模块依赖并验证
sudo depmod -a
sudo modprobe lustre
lctl version
```

### 3.6 安装后验证（所有路径通用）

```bash
# 1) 加载 Lustre 内核模块（无输出即为成功）
sudo modprobe lustre

# 2) 验证模块与用户态工具版本
lctl version          # 期望输出 2.16.x / 2.17.x 版本号
lfs --version         # 用户态工具版本

# 3) 确认模块已加载
lsmod | grep lustre
cat /sys/module/lustre/version
```

| 命令 | 期望结果 |
| --- | --- |
| `modprobe lustre` | 无报错（报错先 `dmesg \| tail` 看原因） |
| `lctl version` | 输出版本号 |
| `lfs --version` | 输出版本号 |

> **排障提示**：`modprobe lustre` 若报 `Invalid module format` 或 `Unknown symbol`，绝大多数原因是**内核头文件与运行内核不一致**（见 §3.1 第 2 步）。DKMS 安装的模块会在内核升级后自动重编，但需要新内核对应的 `linux-headers`，因此建议按 §1.4 锁定内核。

---

## 4. Ubuntu 24.04 服务端源码编译（MGS/MDS/OSS 角色）

> **与客户端不同，Ubuntu 24.04 没有官方预编译服务端 .deb，MGS/MDS/OSS 角色必须从源码编译**，这是 Ubuntu 与 RHEL 系（官方预编译 RPM）的最大差异。以下命令默认在 **Ubuntu 24.04（noble，内核 6.8 GA）** 上执行；Lustre 版本统一为 **2.16.x / 2.17.x**，**不使用 2.15**（2.15 与 kernel 6.x 不兼容）。

---

Lustre 服务端代码与 Linux 内核深度耦合，在 Ubuntu 24.04 上只能基于**当前运行内核的头文件**编译内核模块。核心链路为：确认内核/头文件 → 安装编译依赖 → 选定后端（ZFS 或 ldiskfs）→ `git clone` → `autogen.sh` → `configure` → `make debs` → `dpkg -i` → `depmod` → `modprobe` 验证。

> 本文第 4 章所有命令均在服务端节点（mds1 / oss1）执行；客户端无需编译服务端（见第 3 章）。编译耗时较长（单机通常 2~6 小时，视 CPU/内存而定），属正常现象。

### 4.1 确认内核与头文件

内核头文件必须与**正在运行的内核严格一致**，这是模块能否加载的第一前提。

```bash
# 1) 查看当前内核版本，应为 6.8.0-XX-generic（GA 内核，见第 1.4 节）
uname -r

# 2) 安装与运行内核严格一致的头文件
sudo apt update
sudo apt install -y linux-headers-$(uname -r)

# 3) 复核：已安装的头文件版本必须与 uname -r 完全一致
apt list --installed | grep linux-headers
ls -l /lib/modules/$(uname -r)/build    # 应指向 /usr/src/linux-headers-6.8.0-XX-generic
```

| 检查项 | 期望结果 | 说明 |
| --- | --- | --- |
| `uname -r` | `6.8.0-XX-generic` | GA 内核，**不要用 HWE 内核** |
| `linux-headers-$(uname -r)` | 版本与 `uname -r` 完全一致 | 不一致是 `Invalid module format` 的头号原因 |
| `/lib/modules/$(uname -r)/build` | 符号链接指向 headers 目录 | `make debs` 默认按此目录定位内核头文件 |

**内核线选择**：Ubuntu 24.04 默认 GA 内核 **6.8.x** 是 Lustre 2.16 的官方测试内核（2.16 ChangeLog 将 `6.8.0-38`、`6.10.0-15` 列为 server primary kernels）。**HWE 内核不建议**——HWE 迭代快、生命周期短（约 6 个月），Lustre 模块编译与加载跟不上。

```bash
# 强烈建议：锁定内核，防止 apt upgrade 自动换内核导致模块失效
sudo apt-mark hold linux-image-generic linux-headers-generic
# 若误装了 HWE 内核，先切回 GA（见 §6 第 2 条）
```

### 4.2 安装编译依赖

```bash
sudo apt update
sudo apt install -y build-essential gcc make flex bison pkg-config \
  zlib1g-dev libssl-dev libmount-dev libyaml-dev libnl-3-dev libnl-genl-3-dev \
  libkeyutils-dev libreadline-dev libkrb5-dev swig libtool autoconf \
  python3-dev dpkg-dev
```

| 依赖 | 用途 |
| --- | --- |
| `build-essential` / `gcc` / `make` | 编译工具链 |
| `flex` / `bison` | 生成解析器代码（Lustre 用户态与内核模块构建需要） |
| `pkg-config` / `libtool` / `autoconf` | `autogen.sh` 与 `configure` 需要 |
| `zlib1g-dev` / `libssl-dev` / `libkeyutils-dev` / `libkrb5-dev` / `libreadline-dev` | 各类库绑定（KRB5 用于 GSS 认证，按需） |
| `libmount-dev` / `libyaml-dev` / `libnl-3-dev` / `libnl-genl-3-dev` | 挂载与 LNet 用户态配置支持 |
| `swig` / `python3-dev` | Python 绑定 |
| `dpkg-dev` | 提供 `dpkg-buildpackage`，`make debs` 打包必需 |

**gcc 版本说明**：Ubuntu 24.04 默认 gcc 为 **gcc-13**，与官方 6.8 内核构建工具链一致，**保持默认即可**。LU-18010 中官方也确认需选择 gcc-13。若机器上装了多个 gcc 版本需要切换：

```bash
gcc --version
sudo update-alternatives --config gcc    # 选择 gcc-13
```

> 若 `configure` / `make debs` 提示缺少其他开发库，按报错信息 `apt search <库名>` 补齐即可，常见补充项：`quilt`（旧分支 `make debs` 仍可能依赖）、`libaio-dev`、`libselinux-dev`。

### 4.3 后端选择（二选一）

服务端存储后端有两条路线，**编译前必须确定**，configure 参数不同：

| 对比项 | 路线 A：ZFS 后端 | 路线 B：ldiskfs（patchless） |
| --- | --- | --- |
| 内核补丁 | **完全不需要**（ZFS 模块来自 zfs-dkms，与 Lustre 无关） | 不需要补丁内核，但需要 LU-18010 的 **ext4 源码链接法**（见 §4.3.2） |
| 额外组件 | Ubuntu 自带 ZFS 2.2.x（`zfsutils-linux` + `zfs-dkms`） | Whamcloud 定制 e2fsprogs + `linux-source-6.8.0` |
| target 设备写法 | `<zpool>/<dataset>`（如 `lustre-mdt/mdt`） | 块设备路径（如 `/dev/sdc`） |
| configure 参数 | `--with-zfs --disable-ldiskfs` | 默认（ldiskfs 自动启用），可选 `--without-zfs` |
| 结论 | **推荐**，免补丁、免源码链接，最省心 | 与 04 指南（RHEL 系）命令最接近，但要多做两步准备 |

> 两条路线的 **ZFS 版本要求**：官方支持矩阵中 2.16/2.17 对应 ZFS 2.1.15 / 2.3.4（测试版本），Ubuntu 24.04 自带的 ZFS 2.2.x 一般可直接编译使用；如遇版本兼容报错，以支持矩阵与官方最新说明为准。

#### 4.3.1 路线 A：ZFS 后端（推荐，无补丁内核）

ZFS 后端完全不涉及 ext4/内核源码：Lustre 的 ZFS OSD 直接使用发行版 ZFS 模块与用户态工具，因此**不需要补丁内核，也不需要任何源码链接**。

```bash
# 1) 安装 ZFS（Ubuntu 24.04 自带 ZFS 2.2.x；zfs-dkms 按当前内核自动编译模块）
sudo apt install -y zfsutils-linux zfs-dkms
sudo modprobe zfs
zpool version        # 查看已加载的 ZFS 版本

# 2) 安装 zfs 开发头文件位置确认（configure 探测用）
ls /usr/src | grep zfs    # 期望形如 /usr/src/zfs-2.2.x
```

> **configure 要点**：加 `--with-zfs`（configure 会自动探测 zfs-dkms 安装的头文件目录）；若探测不到，可显式指定 `--with-zfs=/usr/src/zfs-<版本>`。同时加 `--disable-ldiskfs`，跳过对 ext4 源码的依赖。

**创建 target**（详见 §5.4 的完整示例）：

```bash
# 先建 zpool（推荐，可控制池属性），再 mkfs 建 dataset；设备名用 zpool/name 形式
sudo zpool create lustre-mdt /dev/sdc
sudo mkfs.lustre --fsname=lustre --mdt --index=0 \
    --mgsnode=192.168.10.11@tcp0 --backfstype=zfs lustre-mdt/mdt
```

#### 4.3.2 路线 B：ldiskfs（patchless，LU-18010 ext4 链接法）

ldiskfs 模块编译需要 **ext4 内核源码**，而 Ubuntu 的 `linux-headers` 里只有编译产物、没有源码。官方 JIRA **LU-18010** 给出了已验证的简化做法：把内核源码包里的 `fs/ext4` 目录**符号链接**进 headers，即可在**不编译整个内核**的情况下编译出 ldiskfs 服务端模块。需要两步准备：

**（a）编译安装 Whamcloud 定制 e2fsprogs**

ldiskfs 依赖 Whamcloud 定制 e2fsprogs（版本号带 `wc` 后缀），否则 `mkfs.lustre` 无法创建带 Lustre 特有 feature 的文件系统。有两种获得方式：

```bash
# 方式一：源码编译（Debian 系打包流程，来源：Whamcloud 官方 wiki）
git clone https://review.whamcloud.com/tools/e2fsprogs
# 或 git clone git://git.whamcloud.com/tools/e2fsprogs.git
cd e2fsprogs
git checkout master-lustre           # Whamcloud 定制分支（版本号带 wc 后缀）
sed -i 's/ext2_types-wrapper.h$//g' lib/ext2fs/Makefile.in
./configure
dpkg-buildpackage -b -us -uc         # 生成的 .deb 在上一级目录（部分教程用 make deb，以实际为准）
cd ..
sudo dpkg -i *.deb; sudo dpkg -i *.deb   # 有相互依赖，重复执行一次确保装全
e2fsprogs -V                         # 确认版本带 wc 后缀

# 方式二：下载 Whamcloud 预编译 deb（若有，以 downloads.whamcloud.com 实际目录为准）
```

> ⚠️ 定制 e2fsprogs 会覆盖系统原生 e2fsprogs（`mkfs.ext4` 等命令仍可用）。覆盖前建议 `dpkg -l | grep e2fsprogs` 记录原版本号，便于回退。如不想覆盖，可在编译机上用其构建，仅把 `mke2fs`/`e2fsprogs` 相关 deb 分发到服务端安装。

**（b）把内核源码的 ext4 目录链接进 headers（LU-18010 技巧）**

```bash
# 1) 获取与运行内核同系列的源码包（linux-source-6.8.0；小版本尽量贴近 uname -r）
sudo apt install -y linux-source-6.8.0
# 或手动下载：
# wget http://security.ubuntu.com/ubuntu/pool/main/l/linux/linux-source-6.8.0_*.deb
# sudo dpkg -i linux-source-6.8.0_*.deb

# 2) 解压源码包（生成 /usr/src/linux-source-6.8.0/linux-source-6.8.0/）
cd /usr/src/linux-source-6.8.0
sudo tar xjf linux-source-6.8.0.tar.bz2

# 3) 备份 headers 中原生 ext4 目录，并把源码 ext4 链接过去（LU-18010 官方验证可行）
KVER=$(uname -r)
sudo mv /usr/src/linux-headers-${KVER}/fs/ext4 /usr/src/linux-headers-${KVER}/fs/ext4.orig
sudo ln -s /usr/src/linux-source-6.8.0/linux-source-6.8.0/fs/ext4 \
           /usr/src/linux-headers-${KVER}/fs/ext4

# 4) 验证链接生效
ls -l /usr/src/linux-headers-${KVER}/fs/ext4
```

> **说明**：`ext4.orig` 是原生目录的备份；编译完成后可删除链接并 `sudo mv ext4.orig ext4` 还原，恢复 headers 原貌。`linux-source-6.8.0` 与运行内核的小版本（如 `6.8.0-31` vs `6.8.0-45`）允许有少量差异，同系列一般可直接编译；若 ldiskfs 补丁应用失败，优先找与 `uname -r` 完全一致的小版本源码（可查 http://security.ubuntu.com/ubuntu/pool/main/l/linux/ 目录）。

### 4.4 编译安装（make debs）

```bash
# 1) 获取源码并检出目标分支
git clone https://github.com/lustre/lustre-release.git
cd lustre-release
git checkout b2_16          # 2.16.x 稳定线；或 git checkout master（2.17 开发线）

# 2) 生成 configure
sh autogen.sh

# 3) configure——按 §4.3 选定的后端二选一
#    —— 路线 A：ZFS 后端 ——
./configure --enable-server --with-zfs --disable-ldiskfs \
    --with-linux=/usr/src/linux-headers-$(uname -r)
#    —— 路线 B：ldiskfs（需先完成 §4.3.2 的定制 e2fsprogs 与 ext4 链接）——
# ./configure --enable-server --with-linux=/usr/src/linux-headers-$(uname -r)
#    若机器已装 zfs 但本次不用，可追加 --without-zfs 避免误启用

# 4) 打包（Ubuntu 用 debs，等价 rpm 的 rpms；耗时较长属正常）
make debs -j$(nproc)
ls debs/                    # 查看实际产出的 .deb 包名

# 5) 安装（两选一：dpkg -i 安装打包产物，或 make install 直接装本机）
sudo dpkg -i debs/*.deb
# 或
# sudo make install

# 6) 重建模块依赖并验证加载
sudo depmod -a
sudo modprobe libcfs && sudo modprobe lustre
lctl version                # 期望输出 2.16.x / 2.17.x
```

> **要点提示**：
> - 实际 .deb 包名以 `debs/` 目录产出为准，常见包括 `lustre-server-modules-<内核>`、`lustre-server-utils`、`lustre-osd-ldiskfs` / `lustre-osd-zfs`、`lustre-tests` 等；
> - 服务端所有节点（mds1/oss1）必须安装**同一套** debs、使用**同一内核**，否则目标注册/挂载会出现版本不一致；
> - `b2_16` 分支在 Ubuntu 24.04 上若 `autogen.sh` / `make debs` 报缺 `dpatch` 等 Ubuntu 适配问题，参考 LU-18010 中提到的 cherry-pick 补丁（review.whamcloud.com 的 +54216 / +55537，是否仍需以 JIRA 最新状态为准）；
> - 换内核后必须重编：先确认新内核的 `linux-headers-$(uname -r)` 已装，再重复本小节。

### 4.5 常见编译报错

| 现象 | 原因 | 解决 |
| --- | --- | --- |
| `modprobe libcfs` 报 `Invalid module format`，或 `module libcfs: .gnu.linkonce.this_module section size must match the kernel's built struct module size at run time` | 模块针对的内核头文件与**运行内核不匹配**（编译后被换内核/头文件版本不一致） | `sudo apt install --reinstall linux-headers-$(uname -r)`，确认 `uname -r` 与 `/lib/modules/$(uname -r)/build` 指向一致后**重编并重装** |
| ldiskfs 编译失败：找不到 `.../fs/ext4`、或 `patch ... FAILED`（`N out of M hunks FAILED`） | 未做 §4.3.2 的 ext4 链接；或 linux-source 小版本与内核差异过大 | 完成链接；换与 `uname -r` 一致的小版本源码包重试 |
| `configure` 报 `ZFS ... not found` / 找不到 zfs | `--with-zfs` 未探测到 zfs-dkms 头文件 | `ls /usr/src \| grep zfs`，显式 `--with-zfs=/usr/src/zfs-<版本>` |
| `make debs` 报缺 `dpkg-buildpackage` / `dpatch` | 未装打包工具；或 b2_16 分支仍依赖 dpatch | `sudo apt install -y dpkg-dev`（必要时 `sudo apt install -y dpatch`） |
| gcc 相关编译错误 | 编译器版本与内核工具链不匹配 | `sudo update-alternatives --config gcc` 切到 gcc-13，`make clean` 后重编 |

---

## 5. 后续部署流程（Ubuntu 与 RHEL 系差异点）

MGS/MDT/OST 的创建、挂载、LNet、客户端挂载等**核心命令在 Ubuntu 上与 `docs/04-lustre-虚拟机部署指南.md` 完全一致**：`mkfs.lustre`、`mount -t lustre`、`lctl`、`lfs`、`lnetctl` 用法相同。本节只列**差异与要点**，完整流程请对照 04 指南执行。

### 5.1 与 RHEL 系指南（04 指南）的差异总览

| 环节 | RHEL 系（04 指南） | Ubuntu 24.04 | 差异要点 |
| --- | --- | --- | --- |
| 软件安装 | `dnf/yum` + rpm | `apt` + dpkg | 服务端已通过 `make debs` 产出 .deb（§4.4） |
| 防火墙 | `firewalld` | **`ufw`** | `sudo ufw allow 988/tcp` |
| 时间同步 | chrony（dnf 安装） | chrony（**apt** 安装） | 命令相同，包管理不同 |
| SELinux | 需 disabled | 无 SELinux（默认 AppArmor） | 一般无需处理 |
| LNet 配置 | `lnetctl` | `lnetctl` | **完全相同** |
| 目标创建 | ldiskfs 块设备路径 | ZFS：`<zpool>/<dataset>`；ldiskfs：块设备路径 | 见 §5.4 |
| 开机自启 | fstab / systemd | fstab（`_netdev`）/ systemd（Ubuntu 由 systemd 管理挂载时序） | 见 §5.6 |

### 5.2 3 节点拓扑与基础准备

以 **mds1（MGS+MDS）/ oss1（OSS）/ client1（Client）** 三节点为例，网络沿用 04 指南的 `192.168.10.0/24`：

| 节点 | IP | 角色 | 磁盘规划 |
| --- | --- | --- | --- |
| mds1 | 192.168.10.11 | MGS + MDS | `/dev/sdb` = MGT、`/dev/sdc` = MDT |
| oss1 | 192.168.10.12 | OSS | `/dev/sdb` = OST0（可加 `/dev/sdc` = OST1） |
| client1 | 192.168.10.13 | Client | 无需数据盘 |

所有节点基础准备（命令与 04 指南 §5.1 对应，差异仅在包管理）：

```bash
# 1) 主机名（每台分别执行）
sudo hostnamectl set-hostname mds1       # oss1 / client1 各自设置

# 2) /etc/hosts 全节点一致
cat >> /etc/hosts <<EOF
192.168.10.11  mds1
192.168.10.12  oss1
192.168.10.13  client1
EOF

# 3) 时间同步（关键！全节点同一时间源，否则客户端会被 evict，见 §6 第 8 条）
sudo apt update && sudo apt install -y chrony
sudo systemctl enable --now chrony
chronyc sources -v                        # 期望出现 ^* 同步标记

# 4) 防火墙放行 LNet（ufw；实验环境亦可直接不启用）
sudo ufw allow 988/tcp
sudo ufw enable
```

### 5.3 LNet 配置（与 04 指南相同）

```bash
# 所有节点（含客户端）执行；--if 填实际网卡名（ip link 查看）
sudo lnetctl lnet configure
sudo lnetctl net add --net tcp0 --if <实际网卡名>
sudo lnetctl net show
sudo lctl list_nids       # 期望 192.168.10.1X@tcp0

# 连通性验证（服务端之间、客户端→MGS）
sudo lnetctl ping 192.168.10.11@tcp0
```

### 5.4 创建与挂载 MGS/MDT/OST（以 ZFS 后端为例）

本示例采用**先 `zpool create` 再 `mkfs.lustre`** 的推荐方式（官方 wiki 建议显式建池，可控制 `ashift`、`cachefile` 等池属性）；也可直接把块设备交给 `mkfs.lustre --backfstype=zfs` 自动建池，此时池名/数据集名由 Lustre 自动生成，以 mkfs 输出为准。

**mds1（MGS + MDS）**：

```bash
# MGT：独立盘 /dev/sdb；MDT：独立盘 /dev/sdc
sudo zpool create lustre-mgt /dev/sdb
sudo zpool create lustre-mdt /dev/sdc

sudo mkfs.lustre --fsname=lustre --mgs --backfstype=zfs lustre-mgt/mgt
sudo mkfs.lustre --fsname=lustre --mdt --index=0 \
    --mgsnode=192.168.10.11@tcp0 --backfstype=zfs lustre-mdt/mdt

sudo mkdir -p /mnt/mgt /mnt/mdt
sudo mount -t lustre lustre-mgt/mgt /mnt/mgt
sudo mount -t lustre lustre-mdt/mdt /mnt/mdt
```

**oss1（OSS）**：

```bash
# 每个 OST 一块独立盘：/dev/sdb = OST0
sudo zpool create lustre-ost0 /dev/sdb
sudo mkfs.lustre --fsname=lustre --ost --index=0 \
    --mgsnode=192.168.10.11@tcp0 --backfstype=zfs lustre-ost0/ost

sudo mkdir -p /mnt/ost0
sudo mount -t lustre lustre-ost0/ost /mnt/ost0
```

> **追加 OST**：oss1 加盘 `/dev/sdc`，`zpool create lustre-ost1 /dev/sdc` → `mkfs.lustre --ost --index=1 ...` → `mount -t lustre lustre-ost1/ost /mnt/ost1`，其他节点无需改动。
>
> **ldiskfs 路线的唯一区别**：不建 zpool、不加 `--backfstype=zfs`，设备直接用块设备路径，即与 04 指南完全相同的写法：
> ```bash
> # sudo mkfs.lustre --fsname=lustre --mdt --index=0 --mgsnode=192.168.10.11@tcp0 /dev/sdc
> # sudo mount -t lustre /dev/sdc /mnt/mdt
> ```
>
> **生产/HA 建议**：按官方 wiki 显式建池时加 `-O canmount=off -o multihost=on` 等属性（并注意 `cachefile` 的导入方式，见 §5.6）；实验环境使用默认参数即可。

**服务端验证（mds1 上）**：

```bash
sudo lctl dl
# 期望多行：MGS、MDT0000 均 UP；OSS 注册后自动出现 lustre-OST0000-osc-MDT0000 行
sudo lctl get_param mgs.MGS.live.*
```

### 5.5 客户端挂载（client1）

```bash
# 客户端安装见第 3 章（DKMS 仓库或源码编译均可），然后：
sudo modprobe lustre && lctl version
# LNet（§5.3）就绪后挂载：
sudo mkdir -p /mnt/lustre
sudo mount -t lustre 192.168.10.11@tcp0:/lustre /mnt/lustre
lfs df -h
```

> 挂载后的条带化设置与 IO 验证（`lfs setstripe`、`dd`、`fio`）与 04 指南 §5.9 完全一致，此处不重复。

### 5.6 开机自启（fstab / systemd）

Ubuntu 的挂载时序由 **systemd** 管理，与 RHEL 系机制相同，关键点是**服务端目标与客户端挂载在 fstab 中必须写 `_netdev`**，避免开机时网络/模块未就绪导致挂载失败。

| 节点 | 持久化内容 | /etc/fstab 写法示例 |
| --- | --- | --- |
| mds1 | MGT / MDT | `lustre-mgt/mgt /mnt/mgt lustre defaults,_netdev 0 0`；`lustre-mdt/mdt /mnt/mdt lustre defaults,_netdev 0 0` |
| oss1 | OST | `lustre-ost0/ost /mnt/ost0 lustre defaults,_netdev 0 0` |
| client1 | 客户端挂载 | `192.168.10.11@tcp0:/lustre /mnt/lustre lustre defaults,_netdev 0 0` |

> **ldiskfs 后端**：fstab 第一列写块设备路径或 `/dev/disk/by-uuid/<UUID>`（推荐 UUID，避免设备名漂移，见 04 指南 §6.6）。

**ZFS 相关注意**：

```bash
# Ubuntu 安装 zfsutils-linux 后默认启用 zfs-import-cache / zfs-mount 服务，开机自动导入池
systemctl status zfs-import-cache zfs-mount

# 若建池时用了 -o cachefile=none，池不会写入缓存文件，开机无法自动导入；改为写入缓存：
# sudo zpool set cachefile=/etc/zfs/zpool.cache <pool>

# 服务端目标必须在 MGS 之后挂 MDT/OST；fstab 的 _netdev 交由内核等待，一般无需额外处理
```

**LNet 开机自启**：使用 `lnetctl` 动态配置的节点需自建 systemd unit（思路同 04 指南 §5.10）：

```bash
sudo tee /etc/systemd/system/lnet-setup.service >/dev/null <<'EOF'
[Unit]
Description=LNet dynamic setup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/usr/sbin/lnetctl lnet configure
ExecStart=/usr/sbin/lnetctl net add --net tcp0 --if <实际网卡名>

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl enable --now lnet-setup
```

---

## 6. Ubuntu 特有常见问题与排查

> 排障总原则与 04 指南一致：`dmesg | tail`、`journalctl -xe` 日志先行；再看 `lctl dl` 目标状态；最后查网络（`lnetctl ping`、`nc -vz <ip> 988`）。

| # | 现象 | 原因 | 排查 | 解决 |
| --- | --- | --- | --- | --- |
| 1 | `modprobe libcfs` / `modprobe lustre` 报 `Invalid module format`，或 `module struct size mismatch`（`.gnu.linkonce.this_module section size must match ...`） | 内核头文件与**运行内核不匹配**（编译后被换内核、或 headers 版本不一致） | `uname -r`；`apt list --installed \| grep linux-headers`；`ls -l /lib/modules/$(uname -r)/build` | `sudo apt install --reinstall linux-headers-$(uname -r)`，确认版本一致后**重编并重装**模块（§4.4） |
| 2 | HWE 内核导致客户端模块不可用（`-azure`/`-hwe` 内核下 DKMS 编译失败或模块加载失败） | 模块仓库只覆盖 GA 6.8 线，HWE 内核适配跟不上 | `uname -r`（是否带 `-azure`/`-hwe`） | 切回 GA 内核：`sudo apt install -y linux-image-generic linux-headers-generic`，移除 HWE 元包（如 `sudo apt remove linux-image-azure`），重启后在 GRUB 高级选项中选 6.8 GA 内核启动；确认后按 §4.1 锁定内核 |
| 3 | DKMS 模块（zfs-dkms / lustre）加载报 `Required key not available` / `Operation not permitted` / `module verification failed` | **Secure Boot** 拒绝未签名模块 | `mokutil --sb-state`（`SecureBoot enabled` 即命中） | 实验环境直接在 UEFI 固件中**关闭 Secure Boot**；或注册 MOK：`sudo mokutil --import /var/lib/shim-signed/mok/MOK.der`，重启按提示录入密码后 `mokutil --list-enrolled` 确认 |
| 4 | `apt install lustre-client-utils` 失败，提示 `GLIBC_2.38 not found` | 仓库包基于 Ubuntu 24.04（glibc 2.39）构建，旧系统 glibc 版本不够 | `ldd --version \| head -1` | Ubuntu 24.04 自带 glibc **2.39 ≥ 2.38**，满足要求；在 22.04 等旧系统上装 noble 仓库包才会报此错——换对应发行版的仓库或用源码编译 |
| 5 | `mkfs.lustre` 失败，提示 e2fsprogs 版本过旧 / 缺少 ldiskfs 特性（`e2fsprogs too old` / `unknown feature`） | 用的是系统原生 e2fsprogs，需要 **Whamcloud 定制版**（带 `wc` 后缀） | `e2fsprogs -V`（检查版本号） | 按 §4.3.2 编译安装定制 e2fsprogs（或安装预编译 deb），确认版本带 `wc` 后缀后重试 |
| 6 | `modprobe lustre` 提示 `Module lustre not found` | `make install` / `dpkg -i` 未执行，或 `depmod` 未运行 | `find /lib/modules/$(uname -r) -name '*lustre*'`；`find /lib/modules/$(uname -r) -name 'libcfs*'` | 执行 `sudo dpkg -i debs/*.deb`（或 `sudo make install`），然后 `sudo depmod -a`，再 `modprobe libcfs && modprobe lustre` |
| 7 | 编译报 gcc 版本相关错误（如 `error: 'struct module' has no member ...`） | 编译器版本与内核/头文件工具链不匹配 | `gcc --version`；`sudo update-alternatives --config gcc` 查看候选 | 切换到 gcc-13（Ubuntu 24.04 默认，与 6.8 内核一致），`make clean` 后重编 |
| 8 | 客户端被 evict（`LustreError: ... timed out` / `no recovery` / `evicted by ...`） | 节点时钟偏差超过恢复窗口，服务端判定客户端失联并驱逐 | `timedatectl`；`chronyc sources -v`（应出现 `^*`） | 全集群 chrony 指向同一时间源；`sudo systemctl restart chrony`；被驱逐后 `umount` / `mount` 恢复 |
| 9 | LNet 连不通（`lnetctl ping` 超时 / 客户端挂载报 `Can't contact MGS` / `Connection timed out`） | NID 拼写错误、网卡名错误、防火墙未放行 988/tcp | 两端 `sudo lctl list_nids` 对比；`sudo lnetctl net show`；`nc -vz 192.168.10.11 988`；`sudo ufw status` | 统一网络名与 NID（默认 `tcp0`）；`--if` 填实际网卡；`sudo ufw allow 988/tcp` |

---

## 7. 参考资料

| 资料 | 地址 / 位置 | 用途 |
| --- | --- | --- |
| Lustre 官方 Wiki | https://wiki.lustre.org/ | 首页（Quick Start、Building/Compiling Lustre 系列、Lustre Manual 索引） |
| Ubuntu 源码编译 walk-thru | https://wiki.whamcloud.com/display/PUB/Build+Lustre+MASTER+with+Ldiskfs+on+Ubuntu+20.04.1+from+Git | 源码编译完整步骤（作者即 LU-18010 报告人，步骤在 24.04 上仍基本适用） |
| LU-18010（JIRA） | https://jira.whamcloud.com/browse/LU-18010 | Ubuntu 24.04 服务端构建：**ext4 链接法**（§4.3.2）出处与后续补丁 |
| Whamcloud 支持矩阵 | https://wiki.whamcloud.com/display/PUB/Lustre+Support+Matrix | 版本 × 发行版 × 内核 × ZFS/e2fsprogs 版本权威对照 |
| Whamcloud 下载站 | https://downloads.whamcloud.com/ | Lustre 包/SRPM 与 e2fsprogs 预编译包（以实际目录为准） |
| e2fsprogs 源码 | `git clone https://review.whamcloud.com/tools/e2fsprogs`（或 `git://git.whamcloud.com/tools/e2fsprogs.git`） | Whamcloud 定制 e2fsprogs（分支 `master-lustre`） |
| Lustre 源码仓库 | `git clone https://github.com/lustre/lustre-release.git` | 服务端源码（分支 `b2_16` / `master`） |
| Azure Managed Lustre 客户端文档 | Microsoft Learn 检索「Azure Managed Lustre」（仓库为 `https://packages.microsoft.com/repos/amlfs-<codename>/`） | 客户端 DKMS 包来源与用法（第 3.2 节） |
| Google Cloud Lustre 文档 | https://cloud.google.com/lustre/docs（具体页面路径以官网为准） | 客户端 DKMS 仓库（`lustre-client-ubuntu-noble`）说明（第 3.3 节） |
| lustre.software 镜像 | https://lustre.software/ | 社区镜像（非官方）：e2fsprogs 镜像、Lustre 客户端/服务端镜像（含 ubuntu2404 client feature release），`trusted=yes` 使用，注意风险 |

**本文档的配套关系**：

- 客户端安装、架构方案与版本选择见本文档第 1~3 章；核心部署命令（`mkfs.lustre`、`mount -t lustre`、`lctl`、`lfs`、`lnetctl`）与 `docs/04-lustre-虚拟机部署指南.md` 一致，本文仅覆盖 Ubuntu 差异点（§5.1 对照表）；
- 概念、`mkfs.lustre` / `lctl` 全参数等权威参考为工作区 `docs/Lustre_Manual_cn_0.0.3.pdf`（官方手册中文版）。
