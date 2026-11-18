# A800 80GB 八卡 GPU Burn 压测操作文档

## 1. 环境信息

| 项目 | 详情 |
|------|------|
| 操作系统 | Ubuntu 22.04 LTS |
| GPU | NVIDIA A800 80GB × 8 |
| 压测工具 | gpu-burn |
| 压测时长 | 30 分钟（1800 秒） |
| 输出目录 | `/root/gpu-burn/` |
| Compute Capability | 8.0 (GA100) |

## 2. 前置检查

```bash
nvidia-smi                        # 检查驱动、8 卡全部可见
nvidia-smi -L | grep A800         # 确认 8 条 A800 记录
nvcc --version                    # 确认 CUDA 已安装
```

## 3. 安装 gpu-burn

```bash
sudo apt update && sudo apt install -y build-essential git
cd /tmp
git clone https://gitee.com/mirrors_wilicc/gpu-burn.git   # 国内用 Gitee 镜像
cd gpu-burn
make COMPUTE=80                    # A800 compute capability 8.0
./gpu_burn -l                      # 验证：确认 8 张 A800
```

> GitHub 连不上时备选：`git clone https://ghproxy.com/https://github.com/wilicc/gpu-burn.git`
>
> 编译报错找不到 nvcc：`export PATH=/usr/local/cuda/bin:$PATH`

## 4. 执行压测

```bash
mkdir -p /root/gpu-burn
cd /tmp/gpu-burn
./gpu_burn -d -tc -m 95% 1800 2>&1 | tee /root/gpu-burn/gpu_burn_output.log
```

| 参数 | 含义 |
|------|------|
| `-d` | 双精度 FP64 |
| `-tc` | Tensor Core |
| `-m 95%` | 95% 显存（约 76GB） |
| `1800` | 30 分钟 |

> 快速验证：先跑 `./gpu_burn -d -tc -m 95% 120`（2 分钟），无异常再跑 30 分钟。
> 单卡测试：`./gpu_burn -d -tc -i 0 1800`

## 5. 压测期间监控（另开终端）

| 终端 | 命令 | 用途 |
|------|------|------|
| 终端 2 | `nvidia-smi --query-gpu=timestamp,name,temperature.gpu,power.draw,utilization.gpu,utilization.memory,memory.used,memory.total --format=csv -l 5 -f /root/gpu-burn/gpu_monitor.csv` | CSV 监控数据 |
| 终端 3 | `dmesg -wT \| grep -iE "nvidia\|error\|xid\|nvrm" \| tee /root/gpu-burn/dmesg_error.log` | 系统错误日志 |
| 终端 4 | `watch -n 2 nvidia-smi` | 实时观察 |

## 6. 结果判定

**正常：** 无报错退出，温度 70-85°C，功耗 ~300W，利用率 99-100%，dmesg 无 Xid 错误。

**异常：**

| 现象 | 可能原因 |
|------|----------|
| 中途报错退出 | 某卡不稳定、显存故障 |
| 温度 > 85°C | 散热不足 |
| 某卡功耗明显偏低 | 该卡故障或降频 |
| 利用率 < 99% | throttle 降频 |
| Xid 错误（13/31/43/48/79 等） | GPU 硬件故障 |

## 7. 收集数据 & 生成报告

```bash
# 收集数据
nvidia-smi -q > /root/gpu-burn/gpu_info_after.txt
dmesg -T | grep -i nvidia > /root/gpu-burn/dmesg_nvidia.log
dmesg -T | grep -i "xid\|error\|fail" > /root/gpu-burn/dmesg_all_error.log

# 生成报告
cd /tmp/gpu-burn && ./generate_report.sh
ls -la /root/gpu-burn/
```

**报告脚本** `/tmp/gpu-burn/generate_report.sh`：

```bash
#!/bin/bash
# 用法：cd /tmp/gpu-burn && ./generate_report.sh
OUTPUT_DIR="/root/gpu-burn"
REPORT="${OUTPUT_DIR}/A800_GPU_Burn_压测报告_$(date +%Y%m%d_%H%M%S).md"

cat > ${REPORT} << EOF
# A800 80GB × 8 卡 GPU Burn 压测报告

**测试时间：** $(date '+%Y-%m-%d %H:%M:%S')
**测试时长：** 30 分钟

## 硬件信息
\`\`\`
$(nvidia-smi -L)
\`\`\`

## 驱动版本
\`\`\`
$(nvidia-smi | head -5)
\`\`\`

## 压测后 GPU 状态
\`\`\`
$(nvidia-smi --query-gpu=index,name,temperature.gpu,power.draw,utilization.gpu,memory.used,memory.total --format=table)
\`\`\`

## 8 卡结果判定
| GPU | 温度(°C) | 功耗(W) | 利用率(%) | 状态 |
|-----|----------|---------|-----------|------|
$(for i in $(seq 0 7); do
    t=$(nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader -i $i)
    p=$(nvidia-smi --query-gpu=power.draw --format=csv,noheader -i $i)
    u=$(nvidia-smi --query-gpu=utilization.gpu --format=csv,noheader -i $i)
    echo "| GPU $i | $t | $p | $u | 正常 |"
done)

## 系统错误日志
\`\`\`
$(cat ${OUTPUT_DIR}/dmesg_all_error.log 2>/dev/null || echo "无错误")
\`\`\`

## 压测原始输出
\`\`\`
$(cat ${OUTPUT_DIR}/gpu_burn_output.log 2>/dev/null || echo "无日志")
\`\`\`

## 结论
（根据实际结果填写）
EOF
echo "报告已生成：${REPORT}"
```

## 8. 注意事项

- A800 八卡满载功耗 2400W+，确保电源和散热充足
- 压测期间勿操作其他 GPU 任务
- `-m 95%` 约使用 76GB 显存，A800 单卡 80GB
- 编译必须指定 `COMPUTE=80`（GA100 架构）
- 压测后保留 `/root/gpu-burn/` 日志归档

## 9. 完整流程总结

```bash
# ===== 1. 前置检查 =====
nvidia-smi && nvidia-smi -L | grep A800 && nvcc --version

# ===== 2. 安装 gpu-burn =====
cd /tmp
git clone https://gitee.com/mirrors_wilicc/gpu-burn.git
cd gpu-burn && make COMPUTE=80

# ===== 3. 创建输出目录 =====
mkdir -p /root/gpu-burn

# ===== 4. 启动监控（另开终端） =====
# 终端 2：nvidia-smi --query-gpu=timestamp,name,temperature.gpu,power.draw,utilization.gpu,utilization.memory,memory.used,memory.total --format=csv -l 5 -f /root/gpu-burn/gpu_monitor.csv
# 终端 3：dmesg -wT | grep -iE "nvidia|error|xid|nvrm" | tee /root/gpu-burn/dmesg_error.log
# 终端 4：watch -n 2 nvidia-smi

# ===== 5. 启动压测 =====
cd /tmp/gpu-burn
./gpu_burn -d -tc -m 95% 1800 2>&1 | tee /root/gpu-burn/gpu_burn_output.log

# ===== 6. 收集数据 =====
nvidia-smi -q > /root/gpu-burn/gpu_info_after.txt
dmesg -T | grep -i nvidia > /root/gpu-burn/dmesg_nvidia.log
dmesg -T | grep -i "xid\|error\|fail" > /root/gpu-burn/dmesg_all_error.log

# ===== 7. 生成报告 =====
cd /tmp/gpu-burn && ./generate_report.sh
ls -la /root/gpu-burn/
```