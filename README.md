# 📚 安装部署文档集

> 日常工作中沉淀的 **软件安装 · 集群部署 · 大模型部署** 实战文档，全部为实际操作记录。

---

## 🗂️ 教程

### ☁️ Ceph 分布式存储 —— `教程/ceph/`

| 文档 | 内容 |
|:---|:---|
| [ceph部署-副本模式](教程/ceph/ceph部署-副本模式.md) | 集群完整部署（副本模式） |
| [ceph-gui-deploy](教程/ceph/ceph-gui-deploy.md) ｜ [ceph-gui-summary](教程/ceph/ceph-gui-summary.md) | Dashboard 图形界面部署与总结 |
| [ceph-add-node](教程/ceph/ceph-add-node.md) | 集群扩容 · 新增节点 |
| [ceph-client-mount](教程/ceph/ceph-client-mount.md) | 客户端挂载 |
| [cephfs-guide](教程/ceph/cephfs-guide.md) | CephFS 文件系统使用 |
| [rgw-guide](教程/ceph/rgw-guide.md) | RGW 对象存储使用 |

### ⚡ Lustre 并行文件系统 —— `教程/lustre/`

**📖 原理与规划**

- [01 · 知识点详解](教程/lustre/01-lustre-知识点详解.md)
- [02 · 研究规划](教程/lustre/02-lustre-研究规划.md)
- [03 · Lustre vs Ceph 对比](教程/lustre/03-lustre-vs-ceph-对比.md)
- [07 · 教学讲解](教程/lustre/07-lustre-教学讲解.md)

**🚀 部署方案**（按目标环境选择）

- [04 · 虚拟机部署指南](教程/lustre/04-lustre-虚拟机部署指南.md)
- [05 · Ubuntu 24.04 部署指南](教程/lustre/05-lustre-ubuntu24.04-部署指南.md)
- [06 · Ubuntu 24.04 三节点部署方案](教程/lustre/06-lustre-ubuntu24.04-3节点-部署方案.md)
- [09 · 单机测试机部署方案](教程/lustre/09-lustre-单机测试机-部署方案.md)
- [10 · 单机虚拟机部署方案](教程/lustre/10-lustre-单机虚拟机-部署方案.md)
- [11 · 单机 192.168.12.160 部署方案](教程/lustre/11-lustre-单机192.168.12.160-部署方案.md)
- [13 · 单机部署总结](教程/lustre/13-lustre-单机部署-总结.md)

**🔧 运维与其他**

- [08 · 新增 OSS 节点与数据盘](教程/lustre/08-lustre-新增OSS节点与数据盘.md)
- [12 · 大模型对比 Hy4-preview / V4-Flash / GLM-5.3-Flash](教程/lustre/12-大模型对比-Hy4-preview-vs-V4Flash-vs-GLM53Flash.md)

### 🖥️ Slurm 集群 —— `教程/slurm集群部署/`

- [Ubuntu-22.04 源码编译部署 Slurm-24.11.6 完整笔记](教程/slurm集群部署/Ubuntu-22.04-源码编译部署Slurm-24.11.6集群完整笔记.md)
- [Ubuntu-22.04 Slurm-24.11.6 集群部署流程（修正完善版）](教程/slurm集群部署/Ubuntu-22.04-Slurm-24.11.6集群部署流程-修正完善版.md)
- [Slurm 集群 10 节点部署文档](教程/slurm集群部署/Slurm集群10节点部署文档.md)
- [slurm-1x86master-12dgxspark 安装指南](教程/slurm集群部署/slurm-1x86master-12dgxspark安装指南.md)
- [机器 IP 地址规划](教程/slurm集群部署/机器ip地址.md)
- 存档：[slurm集群完整流程.pdf](教程/slurm集群部署/slurm集群完整流程.pdf) ｜ [.docx](教程/slurm集群部署/slurm集群完整流程.docx)

### 🛠️ 常用软件安装 —— `教程/` 根目录

| 文档 | 文档 |
|:---|:---|
| [Anaconda 安装](教程/Anaconda安装教程.md) | [Docker 安装部署](教程/docker安装部署.md) |
| [Docker 跑 DeepSeek-V4-Flash-0731](教程/docker跑deepseekv4flash0731.md) | [ComfyUI 部署（四卡）](教程/comfyui部署四卡.md) |
| [Dify 安装部署](教程/Dify安装部署.md) | [RStudio · Ubuntu 26.04](教程/RStudio%20最新版在Ubuntu26.04安装教程.md) |
| [Ubuntu 22.04 科学计算环境依赖](教程/Ubuntu22.04-科学计算环境依赖安装部署.md) | [Ubuntu 密码破解](教程/Ubuntu密码破解教程.md) |
| [芯宇驰 E240 NVMe U.2 4.0 硬盘测试报告](教程/芯宇驰E240%20NVMe%20U.2%204.0硬盘测试报告.md) | |

---

## 🤖 模型部署

- [部署本地模型](模型部署/部署本地模型.md) — Miniconda + vLLM + Open WebUI + ModelScope 模型下载，含代理开关说明

## 🧠 知识 —— `知识/Ceph/`

- [Ceph 核心组件详解](知识/Ceph/Ceph核心组件详解.md) — 三层架构与核心组件原理
- [Lustre 与 Ceph 核心组件详解](知识/Ceph/Lustre与Ceph核心组件详解.md) — 两大存储系统组件对照

## 🌐 代理

- [配置代理](代理/配置代理.md) — Linux 终端代理环境变量设置 / 取消速查

## 🧩 其他

- [AMD MxGPU 现代部署方案（QEMU/KVM + Ubuntu 22.04）](其他/AMD_MxGPU_VMware_ESXi_部署文档.md) — 替代早期 ESXi 方案
- [GPU 虚拟化（AMD）笔记](其他/gpu虚拟化(AMD).md)
- [DeepSeek · GLM · Qwen 视觉多模态模型对比报告](其他/deepseek-vs-glm-vs-qwen-vision-models.md)

---

## 📌 说明

- 环境版本、IP 地址等以各文档内标注为准。
- ⚠️ 部分文档含内网地址，**仅限内部使用，请勿外传**。
- 日常克隆 / 提交 / 推送操作见 [GitHub 仓库使用指南](GitHub仓库使用指南.md)。
