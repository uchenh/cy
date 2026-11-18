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
- [Slurm 集群现场交付上架与现场适配指南](教程/slurm集群部署/Slurm集群现场交付上架与现场适配指南.md) — 10 节点 aiclust 现场上架、改 IP、验收移交全流程
- [slurm-1x86master-12dgxspark 安装指南](教程/slurm集群部署/slurm-1x86master-12dgxspark安装指南.md)
- [机器 IP 地址规划](教程/slurm集群部署/机器ip地址.md)
- 存档：[slurm集群完整流程.pdf](教程/slurm集群部署/slurm集群完整流程.pdf) ｜ [.docx](教程/slurm集群部署/slurm集群完整流程.docx)

### 🏢 现场集群部署（10 节点网络与集群）—— `教程/现场集群部署/`

现场网络与集群部署成套文档，从拓扑到验收共 13 篇（详见 [目录索引](教程/现场集群部署/README.md)）：

- [01 · 总览与拓扑](教程/现场集群部署/01-总览与拓扑.md) ｜ [02 · 业务交换机 S6880](教程/现场集群部署/02-业务交换机-S6880.md) ｜ [03 · 管理交换机 S5120V3](教程/现场集群部署/03-管理交换机-S5120V3.md)
- [04 · 防火墙 F1000-AI-25](教程/现场集群部署/04-防火墙-F1000-AI-25.md) ｜ [05 · 服务器与集群服务](教程/现场集群部署/05-服务器与集群服务.md) ｜ [06 · IB 算力网 MQM8790](教程/现场集群部署/06-IB算力网-MQM8790.md)
- [07 · 客户远程接入 SSLVPN](教程/现场集群部署/07-客户远程接入-SSLVPN.md) ｜ [08 · 部署流程与步骤](教程/现场集群部署/08-部署流程与步骤.md) ｜ [09 · 验证与验收清单](教程/现场集群部署/09-验证与验收清单.md)
- [10 · 附录A 物料与接口适配](教程/现场集群部署/10-附录A-物料与接口适配.md) ｜ [11 · 附录B 现场坑与排障](教程/现场集群部署/11-附录B-现场坑与排障.md) ｜ [12 · 文档一致性核对报告](教程/现场集群部署/12-文档一致性核对报告.md)

### 🛠️ 常用软件安装 —— `教程/` 根目录

| 文档 | 文档 |
|:---|:---|
| [Anaconda 安装](教程/Anaconda安装教程.md) | [Docker 安装部署](教程/docker安装部署.md) |
| [Docker 跑 DeepSeek-V4-Flash-0731](教程/docker跑deepseekv4flash0731.md) | [ComfyUI 部署（四卡）](教程/comfyui部署四卡.md) |
| [Dify 安装部署](教程/Dify安装部署.md) | [RStudio · Ubuntu 26.04](教程/RStudio%20最新版在Ubuntu26.04安装教程.md) |
| [Ubuntu 22.04 科学计算环境依赖](教程/Ubuntu22.04-科学计算环境依赖安装部署.md) | [Ubuntu 密码破解](教程/Ubuntu密码破解教程.md) |
| [芯宇驰 E240 NVMe U.2 4.0 硬盘测试报告](教程/芯宇驰E240%20NVMe%20U.2%204.0硬盘测试报告.md) | |
| [Ceph 副本模式 · 规划与部署](教程/ceph/ceph-副本模式-规划与部署.md) | [Docker 部署记录 · 192.168.18.116](教程/docker部署记录-192.168.18.116.md) |
| [OpenOnDemand 安装部署](教程/OpenOnDemand安装部署.md) | [OpenOndemand 调研手册](教程/OpenOndemand调研手册.md) |
| [A800 GPU Burn 压测操作文档](教程/A800-GPU-Burn压测操作文档.md) | [网络技术培训总结](教程/网络技术培训总结.md) |

### 📄 文档转录（由 Word/PDF 转换，含原图）

| 文档 | 文档 |
|:---|:---|
| [Ceph 部署与使用手册](教程/Ceph部署与使用手册.md) | [Lustre 部署参考手册](教程/Lustre部署参考手册.md) |
| [Slurm 任务调度系统教学](教程/Slurm任务调度系统教学.md) | [Rocky Linux 9.5 安装教程](教程/Rocky_Linux_9.5_安装教程.md) |
| [Ubuntu 22.04 + Kubernetes 1.28 部署](教程/k8s/Ubuntu%2022.04%20单Master节点部署Kubernetes%201.28%20+%20可视化Dashboard.md) | [Kimi K3 四机八卡部署](教程/Kimi_K3_四机八卡分布式推理部署指南.md) |
| [4090/5090 开启 P2P](教程/4090、5090开启p2p教程.md) | [6A15X3 机型 NCCL 测试](教程/6A15X3机型测试NCCL.md) |
| [fabric 驱动性能优化 · NCCL 测试](教程/针对16卡机安装fabric驱动性能提升------测试NCCL.md) | [BIOS 配置 · W790](教程/BIOS配置--W790.md) |
| [AMD 显卡测试报告](教程/AMD显卡测试报告.md) | [标品大模型测试笔记](教程/标品大模型测试笔记.md) |
| [DGX Spark 性能测试报告](教程/DGX%20SPARK性能测试报告.md) | [DGX Spark 调研报告](教程/DGX%20SPARK调研报告.md) |

---

## 🤖 模型部署

- [部署本地模型](模型部署/部署本地模型.md) — Miniconda + vLLM + Open WebUI + ModelScope 模型下载，含代理开关说明
- [海光 BW100 · Qwen3.8 vLLM 部署笔记](模型部署/海光BW100-Qwen3.8-vLLM部署笔记.md)
- [GLM5.2-FP8 并发性能分析报告](模型部署/GLM5.2-FP8-并发性能分析报告.md)
- [4 卡 PRO5000 NCCL 仅 20GB 排障记录](模型部署/4卡PRO5000-NCCL仅20GB排障记录.md)

## 🧠 知识

**Ceph** —— `知识/Ceph/`

- [Ceph 核心组件详解](知识/Ceph/Ceph核心组件详解.md) — 三层架构与核心组件原理
- [Lustre 与 Ceph 核心组件详解](知识/Ceph/Lustre与Ceph核心组件详解.md) — 两大存储系统组件对照
- [Ceph 集群部署研究成果分享 · 讲解指南](知识/Ceph/Ceph集群部署研究成果分享-讲解指南.md)

**Linux 技术要点** —— `知识/Linux/`

- [Linux Ansible 详解](知识/Linux/Linux_Ansible_详解.md)
- [Linux Zabbix 监控系统详解](知识/Linux/Linux_Zabbix_监控系统详解.md)
- [Linux iptables 详解](知识/Linux/Linux_iptables_详解.md)
- [NUMA 技术](知识/Linux/NUMA技术.md)

**前沿资讯** —— `知识/`

- [前沿咨讯](知识/前沿咨讯.md) — AI 服务器基础设施与大模型落地动态

## 🌐 代理

- [配置代理](代理/配置代理.md) — Linux 终端代理环境变量设置 / 取消速查

## 🧩 其他

- [AMD MxGPU VMware ESXi 部署文档](其他/AMD_MxGPU_VMware_ESXi_部署文档.md)
- [DeepSeek · GLM · Qwen 视觉多模态模型对比报告](其他/deepseek-vs-glm-vs-qwen-vision-models.md)
- [Ubuntu sudo/reboot 无效 · 假 systemctl 修复记录](其他/Ubuntu-sudo-reboot无效-假systemctl修复记录.md)
- [Ubuntu 22.04 挖矿木马排查与清理记录](其他/Ubuntu22.04-挖矿木马排查与清理记录.md)
- [GPU 集群 IDC 上架组网与大模型分布式训练 · 性能测试规划书](其他/GPU集群IDC上架组网及大模型分布式训练-性能测试规划书.md)
- [GPU 集群性能测试规划书 V1.2](其他/GPU集群性能测试规划书_V1.2_含吞吐压力.md)
- [SLURM 调度系统介绍](其他/SLURM调度系统介绍.md)
- [算力网络通信全路径](其他/算力网络通信全路径.md)

---

## 📋 变更记录

- **2026-11-18**：归集本地散落的有用文档入库——新增「教程/现场集群部署」成套 13 篇及配套 HPC 部署手册（HTML 可视化门户 + Word 版）、Slurm 现场交付指南、OpenOndemand 调研手册/Ceph 规划/压测/网络培训等教程、4 篇模型部署与性能排障、Ceph 讲解指南与前沿资讯、2 篇 Ubuntu 排障记录（共 26 篇 md + 3 份附件；OpenOnDemand 安装部署手册与远程已有版本内容一致，未重复入库）；补齐 `slurm集群完整流程.pdf` 修复 README 遗留断链；同步更新 README 导航。源目录文件保留，未覆盖仓库已有文档。
- **2026-11-18（Word/PDF 转录）**：将 22 篇本人技术/部署/测试文档由 Word/PDF 转为 Markdown，抽取内嵌图片 220 张至各文档 `assets/` 目录（约 36 MB）；分类入库（教程 14 篇、知识/Linux 4 篇、其他 4 篇），README 增加「文档转录」「Linux 技术要点」导航。按用户确认原样发布（含内网地址，遵循「仅限内部使用」说明）。已排除 HCIP 题库、公司产品资料、个人文件、投标/控标材料及已有 md 版本的文档。源 Word/PDF 保留未改动。

## 📌 说明

- 环境版本、IP 地址等以各文档内标注为准。
- ⚠️ 部分文档含内网地址，**仅限内部使用，请勿外传**。
- 日常克隆 / 提交 / 推送操作见 [GitHub 仓库使用指南](GitHub仓库使用指南.md)。
