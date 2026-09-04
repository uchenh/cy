# AMD MxGPU 现代部署方案（QEMU/KVM + Ubuntu 22.04）

> 文档状态：已替换早期基于 VMware ESXi + FirePro S7150 的旧方案。
> 依据：AMD 官方 *Instinct Virtualization Driver* 文档（mainline-8.6.0.k）中的 Getting started / Host Configuration 章节。
> 适用：近两年新显卡的 GPU 虚拟化（图形 VDI + AI/计算），基于 **QEMU/KVM + Linux**，不再用 VMware ESXi。

---

## 一、现代 MxGPU 支持的新显卡（近两年）
官方明确支持：

| 场景 | 型号 | 说明 |
|---|---|---|
| 图形 VDI | **Radeon PRO V710** | RDNA 3，28GB GDDR6，支持 SR-IOV（图形工作站 VDI 正解） |
| 计算/AI | **Instinct MI210X** | CDNA 2，64GB HBM2e |
| 计算/AI | **Instinct MI300X** | CDNA 3，304 CU，192GB HBM3 |
| 计算/AI | **Instinct MI325X** | CDNA 3，256GB HBM3E |
| 计算/AI | **Instinct MI350X / MI355X** | CDNA 4，288GB HBM3E（MI350X 256CU；MI355X 更高密度） |

来源：[MxGPU 官方文档](https://instinct.docs.amd.com/projects/virt-drv/en/mainline-8.6.0.k/userguides/Getting_started_with_MxGPU.html)

---

## 二、软件栈与下载地址

| 组件 | 作用 | 下载 |
|---|---|---|
| **PF 驱动（gim）** | 宿主主机驱动，SR-IOV PF 管理 | GitHub Releases（.deb/.rpm/源码） |
| **AMD SMI** | 虚拟 GPU 监控/管理 | 随 PF 驱动自动安装（8.3.0K 起） |
| **VF 驱动（ROCm）** | guest VM 内 VF 驱动 | ROCm 安装文档 |

官方驱动下载（PF 驱动 .deb/.rpm 及源码）：
- GitHub `amd/MxGPU-Virtualization` Releases：https://github.com/amd/MxGPU-Virtualization/releases

推荐宿主/guest 发行版：**Ubuntu 22.04、RHEL 9.4**（也可用其他发行版，文档以此为例）。

推荐安装文档（在线更新）：
- 开始虚拟化（含支持的显卡列表）：https://instinct.docs.amd.com/projects/virt-drv/en/mainline-8.6.0.k/userguides/Getting_started_with_MxGPU.html
- 宿主配置：https://instinct.docs.amd.com/projects/virt-drv/en/mainline-8.6.0.k/userguides/Host_configuration.html
- VM 配置：https://instinct.docs.amd.com/projects/virt-drv/en/mainline-8.6.0.k/userguides/VM_setup.html
- GPU 分区：https://instinct.docs.amd.com/projects/virt-drv/en/mainline-8.6.0.k/userguides/GPU_partitioning.html
- XGMI 配置：https://instinct.docs.amd.com/projects/virt-drv/en/mainline-8.6.0.k/userguides/XGMI_configuration.html

---

## 三、部署步骤（Ubuntu 22.04 宿主）

### 1. BIOS 设置
开启：
- **SR-IOV Support**（Advanced → PCI Subsystem Settings）
- **Above 4G Decoding**（同上）
- **PCIe ARI Support**（同上）
- **IOMMU**（Advanced → NB Configuration）
- **ACS Enabled**（NB Configuration，需先开启 AER：Advanced → ACPI Settings → PCI AER Support）

### 2. GRUB 配置
```bash
sudo nano /etc/default/grub
# 修改 GRUB_CMDLINE_LINUX 为（Intel 平台把 amd_iommu 换成 intel_iommu）：
GRUB_CMDLINE_LINUX="modprobe.blacklist=amdgpu iommu=on amd_iommu=on"
```
```bash
sudo update-grub && sudo reboot
```
验证：
```bash
cat /proc/cmdline   # 应包含 modprobe.blacklist=amdgpu iommu=on amd_iommu=on
```
> 说明：`modprobe.blacklist=amdgpu` 让设备从原生驱动解绑，改用 VFIO 才能传给 guest。

### 3. 安装 KVM/QEMU/libvirt
```bash
sudo apt update
sudo apt install qemu-kvm virtinst libvirt-daemon virt-manager -y
sudo usermod -a -G libvirt $(whoami)
```
编辑 `/etc/libvirt/libvirtd.conf`：
```text
unix_sock_group = "libvirt"
unix_sock_rw_perms = "0770"
```
```bash
sudo systemctl restart libvirtd.service
```
> 修改后需重新登录会话使组权限生效。

### 4. 安装 PF 驱动（推荐 .deb 包）
```bash
sudo apt install build-essential dkms autoconf automake
# 从 GitHub Releases 下载 .deb 后：
sudo dpkg -i gim_driver_package.deb
sudo reboot
sudo modprobe gim        # 默认每卡 1 个 VF
```
> 也可从源码编译：`make clean && make all -j && sudo make install && sudo modprobe gim`。
> 新版本 Ubuntu 内核需 gcc-12/g++-12（`sudo apt install gcc-12 g++-12` 并 update-alternatives）。

### 5. 验证 VF 是否出现（按型号查询 PCI ID）
```bash
lspci -d 1002:7461   # Radeon PRO V710
lspci -d 1002:74b5   # Instinct MI300X
lspci -d 1002:74b9   # Instinct MI325X
lspci -d 1002:75b0   # Instinct MI350X
sudo dmesg | grep GIM.*Running   # 确认驱动加载成功
```

### 6. 后续：VM 配置（两条路）
- **简单部署**：默认 VF 设置直接建 VM（官方 VM Setup 页）
- **高级部署**（建 VM 前可选其一或都做）：
  - **GPU Partitioning**：把单个 VF 再切分（SPX/DPX/QPX/CPX），用 `amd-smi set --accelerator-partition=<profile_index>` 动态切换
  - **XGMI 配置**：不同 VM 分配 VF 的组合方式

---

## 四、对比：为什么弃用旧方案

| | 旧方案（S7150 + VMware） | 现代方案（V710/Instinct + KVM） |
|---|---|---|
| 平台 | VMware ESXi | **QEMU/KVM + Linux** |
| 文档 | 停更的老 PDF | **持续更新的官方在线文档** |
| 驱动 | 闭源 VIB，授权受限 | **GitHub 公开 .deb/.rpm + 源码** |
| 收费 | 软件免费但驱动受限 | 全免费、开源为主 |
| 支持新卡 | 否 | 是（近两年卡全支持） |

---

## 五、落地确认清单
- [ ] 显卡型号在受支持列表（V710 / MI210X / MI300X / MI325X / MI350X/MI355X）
- [ ] BIOS 已开启 SR-IOV / Above 4G / ARI / IOMMU / ACS+AER
- [ ] GRUB 已加 `modprobe.blacklist=amdgpu iommu=on amd_iommu=on` 并重启
- [ ] 已安装 libvirt/kvm/qemu 并加入 libvirt 组
- [ ] 已从 GitHub Releases 安装 PF 驱动并用 `lspci -d 1002:xxxx` 确认 VF
- [ ] `dmesg | grep GIM.*Running` 无异常