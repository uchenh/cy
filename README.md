# 安装部署文档集

> 本仓库收录日常工作中的软件安装、集群部署、模型部署及相关知识整理文档，全部为实际操作过程中沉淀的 Markdown / PDF / Word 资料。

## 目录结构

```
安装部署/
├── 教程/
│   ├── ceph/            # Ceph 部署与运维实战（扩容节点、客户端挂载、GUI 部署、CephFS、RGW 等 7 篇）
│   ├── lustre/          # Lustre 分布式并行文件系统（知识点、部署方案、教学讲解等 13 篇）
│   ├── slurm集群部署/    # Slurm 集群部署指南、完整流程笔记及机器 IP 清单（8 篇）
│   └── *.md             # 常用软件安装教程（Anaconda、Dify、Docker、ComfyUI、RStudio 等）
├── 代理/                # 代理环境配置
├── 其他/                # AMD MxGPU、GPU 虚拟化等杂项部署文档
├── 模型部署/            # 本地大模型部署
├── 知识/                # 基础知识整理（Ceph 核心组件等）
└── README.md
```

## 内容概览

### 教程/ceph/
Ceph 集群部署与运维实战：新增节点扩容、客户端挂载、Dashboard GUI 部署与总结、CephFS 使用指南、副本模式部署、RGW 对象存储指南。

### 教程/lustre/
Lustre 文件系统全流程资料：知识点详解、研究规划、Lustre vs Ceph 对比、虚拟机部署指南、Ubuntu 24.04 单机/三节点/新增 OSS 节点等多种部署方案、教学讲解与部署总结。

### 教程/slurm集群部署/
Slurm 集群部署实战：1x86master + 12 DGX Spark 安装指南、10 节点部署文档、Ubuntu 22.04 源码编译部署 Slurm 24.11.6 完整笔记（修正完善版）、集群完整流程（docx/pdf）及机器 IP 地址清单。

### 教程/（根目录）
Anaconda 安装、ComfyUI 四卡部署、Dify 安装部署、Docker 安装与 DeepSeek 容器部署、RStudio 最新版安装、Ubuntu 22.04 科学计算环境依赖、Ubuntu 密码破解等教程。

### 其他
AMD MxGPU on VMware ESXi 部署、GPU 虚拟化（AMD）方案文档。

### 模型部署
本地模型部署实践记录。

### 知识
Ceph 核心组件详解、Lustre 与 Ceph 核心组件对比详解。

### 代理
代理服务配置方法。

## 说明

- 文档以实际部署记录为主，环境版本、IP 地址等以各文档内标注为准。
- 部分文档含有内网地址信息，仅限内部使用，请勿外传。
