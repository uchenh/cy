# 安装部署文档集

> 日常工作中的软件安装、集群部署、模型部署及相关知识整理，全部为实际操作中沉淀的资料（Markdown / Word / PDF）。

## 目录结构

```
cy/
├── 教程/
│   ├── ceph/            # Ceph 部署与运维（扩容、客户端挂载、CephFS、RGW 等）
│   ├── lustre/          # Lustre 分布式文件系统（知识点、部署方案、教学讲解）
│   ├── slurm集群部署/    # Slurm 集群部署与完整流程
│   └── *.md             # 常用软件安装教程（Anaconda、Docker、Dify、ComfyUI、RStudio 等）
├── 模型部署/            # 本地大模型部署（Conda + vLLM + Open WebUI + ModelScope 模型下载）
├── 知识/                # 基础知识整理（Ceph / Lustre 核心组件详解）
├── 代理/                # 代理环境配置
├── 其他/                # AMD MxGPU、GPU 虚拟化等杂项
├── GitHub仓库使用指南.md # 本仓库日常使用速查（克隆 / 提交 / 分支）
└── README.md
```

## 内容概览

- **教程/**：集群与软件部署实战
  - `ceph/`：新增节点扩容、客户端挂载、Dashboard GUI、CephFS、副本模式、RGW 对象存储
  - `lustre/`：知识点详解、Lustre vs Ceph 对比、单机/三节点/新增 OSS 等多种部署方案与教学讲解
  - `slurm集群部署/`：多节点集群安装指南、Ubuntu 源码编译部署 Slurm、完整流程笔记及机器 IP 清单
  - 根目录：Anaconda、Dify、Docker（含 DeepSeek 容器）、ComfyUI 四卡、RStudio、Ubuntu 环境依赖等安装教程，以及 SSD 硬盘测试报告
- **模型部署/**：本地大模型部署记录——Miniconda 环境、vLLM 推理框架、Open WebUI 界面，以及 ModelScope 多模型下载命令清单
- **知识/**：Ceph、Lustre 核心组件详解与对比
- **代理/**：代理服务配置方法
- **其他/**：AMD MxGPU on VMware ESXi、GPU 虚拟化（AMD）方案
- **GitHub仓库使用指南.md**：本仓库克隆与日常提交的分支操作速查

## 说明

- 文档以实际部署记录为主，环境版本、IP 地址等以各文档内标注为准。
- 部分文档含有内网地址信息，仅限内部使用，请勿外传。
