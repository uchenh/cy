# RStudio 最新版在 Ubuntu 26.04 安装教程

> 从零开始，在 Ubuntu 26.04 LTS（Resolute Raccoon）上安装 R 语言与最新版 RStudio Desktop（2026.08.0），含依赖处理与常见问题排查。

| 项目 | 说明 |
|------|------|
| 系统 | Ubuntu 26.04 LTS |
| 软件 | RStudio 2026.08.0 |
| 更新日期 | 2026-08-17 |
| 适用架构 | amd64 |

---

## 一、概述与版本现状

Ubuntu 26.04 LTS（代号 **Resolute Raccoon**，坚毅浣熊）已于 **2026 年 4 月 23 日** 正式发布，是 Canonical 的第 11 个长期支持版本，标准支持至 2031 年 4 月，订阅 Ubuntu Pro 后安全维护可延长至 2036 年[1][2]。该版本基于 Linux 7.0 内核，默认桌面为 GNOME 50（仅支持 Wayland），并沿用 DEB822 格式的软件源配置[1][3]。

RStudio（现由 Posit 公司维护）最新稳定版为 **2026.08.0（代号 Yellow Yarrow）**，发布于 2026 年 8 月 13 日[4]。目前 Posit 官方下载页仅正式列出 Ubuntu 22/24 的安装包，尚未把 26.04 列为官方支持版本，但官方已在 GitHub 上提交了针对 Ubuntu 26.04（resolute）的依赖安装脚本（PR #17482），说明正在为 26.04 做准备[5]。因此本教程采用 **Ubuntu 24 的 .deb 包 + gdebi 自动处理依赖** 的方式，在 26.04 上安装最新版 RStudio。

| 关键信息 | 值 |
|----------|-----|
| Ubuntu 26.04 代号 | Resolute Raccoon |
| 官方源 R 版本 | r-base 4.5.2 |
| RStudio 最新版 | 2026.08.0 |
| 安装方式 | .deb + gdebi |

> **重要提示：关于版本命名**
> RStudio 已由 Posit 公司维护，注意区分 **RStudio Desktop**（本教程目标）与 **Positron**（Posit 推出的另一款基于 VS Code 的独立 IDE）。两者是不同产品，不要混淆[4]。

---

## 二、安装流程总览

整体安装分为四步：更新系统 → 安装 R 语言 → 安装系统依赖 → 安装 RStudio。

```
更新系统 (apt update / upgrade)
     │
     ▼
安装 R 语言 (r-base)
     │
     ▼
安装系统依赖库 (libssl-dev, libclang-dev, libpq5 等)
     │
     ▼
安装 gdebi-core
     │
     ▼
下载 RStudio .deb 包
     │
     ▼
gdebi 安装并自动解析依赖
     │
     ▼
启动 rstudio 验证
     │
     ├─ 成功 → 完成
     │
     └─ 依赖报错 → apt --fix-broken install 修复 → 重新安装
```

> **为什么用 gdebi 而不是 dpkg -i？**
> 直接使用 `dpkg -i` 安装 .deb 包时，dpkg **不会自动下载依赖**，会报 "dependency problems prevent configuration of rstudio"。而 `gdebi` 会自动解析并安装所有依赖，是官方推荐的安装工具[6]。

---

## 三、安装前准备

### 3.1 更新系统

先更新软件包索引并升级系统，确保所有源配置正确。

```bash
sudo apt update
sudo apt upgrade -y
```

### 3.2 确认系统与架构

确认系统版本与 CPU 架构（RStudio 官方提供 amd64 包，Ubuntu 26.04 也支持 arm64 桌面版[1]）。

```bash
lsb_release -a        # 确认是 Ubuntu 26.04
uname -m              # 确认是 x86_64
```

### 3.3 关于软件源格式（DEB822）

Ubuntu 26.04 沿用 **DEB822 格式** 的软件源配置，文件位于 `/etc/apt/sources.list.d/ubuntu.sources`，不再是旧的 `/etc/apt/sources.list`。网上大量基于旧格式的换源教程在 26.04 上不再适用，**换源时务必使用支持 resolute 的镜像**，否则可能引发依赖冲突[3]。

> **t64 库重命名风险**
> 自 Ubuntu 24.04 起，为解决 2038 年问题，大量系统库被重命名为带 `t64` 后缀（如 `libcurl4` → `libcurl4t64`、`libssl3` → `libssl3t64`）。为旧版 Ubuntu 构建的第三方 .deb 若依赖旧库名，会报 "unmet dependencies"。本教程使用最新版 RStudio 并配合 gdebi，可规避大部分此类问题[7]。

---

## 四、安装 R 语言

RStudio 需要 R 3.6.0 及以上版本。Ubuntu 26.04 官方源（universe 组件）中已提供 **r-base 4.5.2**，直接 apt 安装即可，无需第三方源[8]。

### 方式一：官方源安装（推荐，简单稳定）

```bash
sudo apt install r-base r-base-dev
```

其中 `r-base-dev` 用于编译安装 R 源码包（如需要编译某些 CRAN 包）。

### 方式二：CRAN 官方仓库安装（获取更新版本）

如需比 4.5.2 更新的 R 版本，可添加 CRAN 官方 Ubuntu 二进制仓库（已支持 resolute）[9]：

```bash
# 安装签名工具
sudo apt install --no-install-recommends software-properties-common dirmngr

# 添加 CRAN 签名密钥
wget -qO- https://cloud.r-project.org/bin/linux/ubuntu/marutter_pubkey.asc | sudo tee -a /etc/apt/trusted.gpg.d/cran_ubuntu_key.asc

# 添加 CRAN 仓库（26.04 代号为 resolute）
sudo add-apt-repository "deb https://cloud.r-project.org/bin/linux/ubuntu resolute-cran40/"

# 安装 R
sudo apt update
sudo apt install r-base r-base-dev
```

安装完成后验证 R 版本：

```bash
R --version
```

---

## 五、安装 RStudio 系统依赖

RStudio 的 .deb 包依赖以下核心库。根据官方为 Ubuntu 26.04 准备的依赖脚本（PR #17482），核心依赖如下[5]：

| 依赖包 | 作用 |
|--------|------|
| `libssl-dev` | SSL 开发库，RStudio 的 HTTPS 通信依赖 |
| `libclang-dev` | Clang 编译器前端，代码补全与诊断引擎依赖 |
| `libpq5` | PostgreSQL 客户端库 |
| `libcurl4-openssl-dev` | cURL 库，用于 R 包安装与网络请求 |
| `libxml2-dev` | XML 解析库，许多 R 包依赖 |

```bash
sudo apt install libssl-dev libclang-dev libpq5 libcurl4-openssl-dev libxml2-dev
```

> 若 gdebi 在安装时仍提示缺少其他依赖（如 `libgtk-3-0`、`libfuse2` 等），可一并安装：`sudo apt install libgtk-3-0 libfuse2 libsqlite3-dev`，然后继续下一步。

---

## 六、安装 RStudio Desktop

### 6.1 安装 gdebi-core

```bash
sudo apt install gdebi-core
```

### 6.2 下载最新版 .deb 包

从 Posit 官方下载服务器获取最新版（2026.08.0）的 .deb 包。Posit 官方下载页目前列出的是 Ubuntu 22/24 的构建，在 26.04 上可直接使用该构建[10]。

```bash
# 下载 RStudio 2026.08.0 (amd64)
wget https://download1.rstudio.org/electron/jammy/amd64/rstudio-2026.08.0-187-amd64.deb
```

> 访问 [Posit 官方下载页](https://posit.co/download/rstudio-desktop/) 查看当前最新版本号，将 URL 中的 `2026.08.0-187` 替换为最新版本即可。

### 6.3 使用 gdebi 安装

gdebi 会自动解析并安装所有依赖，避免 dpkg 的依赖地狱。

```bash
sudo gdebi rstudio-2026.08.0-187-amd64.deb
```

> 若 gdebi 报 libssl 相关错误，有用户在 Ubuntu 24.0.1 上遇到类似问题，解决办法是给 gdebi 传入 **完整路径**：`sudo gdebi ~/下载/rstudio-2026.08.0-187-amd64.deb`[11]。若仍失败，可先手动安装依赖再重试。

### 6.4 备选安装方式

除官方 .deb 包外，还有以下备选方式（均非 Posit 官方维护，按需选择）：

| 方式 | 命令 | 说明 |
|------|------|------|
| Snap | `sudo snap install rstudio --classic` | 社区维护，非官方，风险自负[12] |
| 社区 PPA | `sudo add-apt-repository "deb https://www.r-tools.ppa.net/project/r-tools-ppa/deb stable main"` | 第三方 PPA，非官方[13] |

---

## 七、启动与验证

安装完成后，在终端输入 `rstudio` 即可启动 IDE，或从应用菜单中找到 RStudio 图标。

```bash
rstudio
```

启动后建议在 R 控制台执行以下命令，验证 R 环境与 RStudio 工作正常：

```r
version
# 测试安装一个常用包
install.packages("ggplot2")
```

> Ubuntu 26.04 桌面仅支持 Wayland，RStudio 作为 X11 应用会通过 **XWayland** 兼容层运行，无需额外配置即可正常显示[1]。

---

## 八、常见问题排查

### 8.1 依赖错误：dpkg: dependency problems prevent configuration of rstudio

原因：直接用 `dpkg -i` 安装，dpkg 不会自动下载依赖。解决办法：

```bash
# 修复损坏的依赖（自动下载并安装缺失依赖）
sudo apt --fix-broken install
```

或改用 gdebi 重新安装[6]。

### 8.2 libssl 相关错误

旧版 RStudio（2022 年及更早）依赖 `libssl1.1`，而 Ubuntu 22.04+ 只有 `libssl3`。解决办法：**务必使用最新版 RStudio**（2026.08.0 已适配 libssl3），不要安装旧版本[6]。

### 8.3 t64 库依赖不满足（libcurl4t64 / libssl3t64）

若遇到此类报错，先执行 `sudo apt --fix-broken install` 修复；确认软件源已正确启用（Ubuntu 升级后可能禁用部分源）[7]。

### 8.4 通用排错流程

```bash
# 1. 修复损坏依赖
sudo apt --fix-broken install

# 2. 安装全部核心依赖
sudo apt install libssl-dev libclang-dev libpq5 libcurl4-openssl-dev libxml2-dev

# 3. 用 gdebi 重新安装 RStudio
sudo gdebi rstudio-*.deb
```

> **非 LTS 版本不支持**
> Posit 官方只构建和测试 Ubuntu LTS 版本的包，非 LTS 版本（如 25.04、25.10）不被官方支持。Ubuntu 26.04 是 LTS，但官方尚未正式列出支持；若遇到问题，可关注 RStudio 官方 PR #17482 的进展[5][14]。

---

## 参考来源

1. Canonical, Ubuntu 26.04 LTS 发布公告（Resolute Raccoon）<https://cn.ubuntu.com/blog/canonical-releases-ubuntu-26-04-lts-resolute-raccoon_cn>
2. Ubuntu 官方 Release Notes 26.04 <https://documentation.ubuntu.com/release-notes/26.04/>
3. 清华 TUNA 镜像 Ubuntu 帮助 DEB822 格式 <https://mirror.tuna.tsinghua.edu.cn/help/ubuntu>
4. Posit, RStudio IDE 发布说明 <https://docs.posit.co/ide/news/>
5. RStudio GitHub PR #17482 <https://github.com/rstudio/rstudio/pull/17482/files>
6. Posit 论坛，依赖问题讨论 <https://forum.posit.co/t/dependency-issues-when-installing-rstudio-on-ubuntu-22-04/153543>
7. ROS distro issue #47577（t64 库重命名）<https://github.com/ros/rosdistro/issues/47577>
8. Ubuntu Packages 页面，r-base 4.5.2 <https://packages.ubuntu.com/resolute/r-base>
9. CRAN Ubuntu 二进制仓库说明 <https://cran.csiro.au/bin/linux/ubuntu/fullREADME.html>
10. Posit, RStudio IDE 用户指南下载链接 <https://docs.posit.co/ide/user/>
11. Posit 论坛，Ubuntu 24.0.1 libssl 错误 <https://forum.posit.co/t/dealing-with-libssl-error-installing-rstudio-on-ubuntu-24-0-1/196977/2>
12. Snap Store, RStudio 页面 <https://snapcraft.io/install/rstudio/ubuntu>
13. GitHub, albersonmiranda/r_tools_ppa <https://github.com/albersonmiranda/r_tools_ppa>
14. Posit 论坛，非 LTS 版本不支持声明 <https://forum.posit.co/t/linux-ubuntu-impossible-to-install-any-package-because-of-dependencies-installation-issue/199354/12>