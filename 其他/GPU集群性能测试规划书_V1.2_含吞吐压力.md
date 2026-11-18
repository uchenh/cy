# GPU集群性能测试规划书_V1.2_含吞吐压力

GPU 集群性能测试规划书

配置核对 → 单机硬件 → GPU/模型 → IB → NCCL → 分布式训练

版本 V1.2｜含吞吐、压力测试与具体命令｜IDC 上架验收


## 1. 测试目的与主线

验证采购硬件齐全可用，单机 CPU/内存/磁盘/GPU 性能正常，IB 与 NCCL 满足多机训练，并通过模型吞吐与分布式训练确认集群可投入使用。

主线：①配置核对 → ②单机硬件(CPU/内存/磁盘) → ③GPU 与模型(含吞吐/压力) → ④IB 网络 → ⑤NCCL → ⑥分布式训练抽测。


## 2. 服务器与网络配置


| 角色 | 数量 | 关键配置 |
|---|---|---|
| 管理节点 | 1 | 2×EPYC 7513；64GB DDR4；960G SSD RAID1；200G IB；2U；800W 1+1 |
| 存储节点 | 1 | 2×EPYC 9124；2×64GB DDR5；4×3.84TB U.2 NVMe；200G IB；PRA244-D12H；1200W 1+1 |
| 计算节点 | 4 | 2×EPYC 9554；16×64GB DDR5(≈1TB)；8×A800 80GB；960G SSD；PGA444-D08；200G IB；2600W 2+2 |
| 网络配套 | 1套 | MQM8700 200G IB 40口；48口千兆+4口万兆交换机；IB线/网线；42U×2；调度系统 |

合计：32×A800 80GB；训练网 200G IB。


### 2.1 节点登记（现场填）


| 主机名 | 角色 | 管理IP | IB IP | 驱动版本 | CUDA | 备注 |
|---|---|---|---|---|---|---|
|  | 管理 |  |  |  |  |  |
|  | 存储 |  |  |  |  |  |
|  | 计算1 |  |  |  |  |  |
|  | 计算2 |  |  |  |  |  |
|  | 计算3 |  |  |  |  |  |
|  | 计算4 |  |  |  |  |  |


## 3. 测试准备

前置：上架理线完成；OS 统一（建议 Ubuntu 22.04）；已装 NVIDIA Driver、CUDA、MLNX_OFED；chrony 同步；SSH 互信；测试目录如 /opt/bench。


| 天 | 内容 |
|---|---|
| D0 | 上架加电、配置核对 A |
| D1 | CPU/内存/磁盘 B + GPU基础 C1 |
| D2 | GPU吞吐/压力/模型 C2 + 单机NCCL |
| D3 | IB + 跨机NCCL |
| D4 | 存储/调度 + 分布式训练 + 汇总签字 |


### 3.1 建议安装的测试软件

# 示例（按实际系统调整）

sudo apt update && sudo apt install -y fio stress-ng iperf3 build-essential git cmake

# CUDA Samples（带宽）: 随 CUDA 或 git clone https://github.com/NVIDIA/cuda-samples

# nccl-tests: git clone https://github.com/NVIDIA/nccl-tests.git && make MPI=1 CUDA_HOME=... NCCL_HOME=...

# gpu-burn: git clone https://github.com/wilicc/gpu-burn.git && make

# stream: 下载/编译 STREAM 基准；memtester: apt install memtester

# perftest(IB): 通常随 MLNX_OFED 提供 ib_write_bw / ib_read_bw / ib_write_lat


## 4. 测试项详表（方法 / 命令 / 预期 / 记录）


### 4.1 A. 配置核对


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| A01 | 清点外观 | 对照订单清点服务器、交换机、IB线、网线；检查外观 | 数量型号一致 |  |  |
| A02 | 上架供电 | 按U位上架贴标；双路电源接不同PDU；加电 | 点亮正常 |  |  |
| A03 | 管理/存储配置 | lscpu free -h lsblk sudo nvme list   #存储节点 | 与订单一致 |  |  |
| A04 | 计算节点配置 | 每台： lscpu \| egrep 'Model name\|CPU\(s\)' free -h nvidia-smi -L | 约128逻辑核；≈1TB；8×A800 |  |  |
| A05 | GPU拓扑 | nvidia-smi topo -m nvidia-smi -q \| grep -E 'Product Name\|VBIOS\|Driver Version' | 形态与采购一致；驱动统一 |  |  |
| A06 | IB状态 | ibstat ibv_devinfo \| head # 看 State/Rate/Port | Active，Rate=200 |  |  |
| A07 | BMC/SSH | 浏览器登BMC；ssh user@管理IP | 带外与SSH可达 |  |  |


### 4.2 B. 单机硬件：CPU / 内存 / 磁盘

计算节点 4 台全做；管理/存储做识别+磁盘（CPU压测可抽测）。


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| B01 | CPU识别 | lscpu cat /proc/cpuinfo \| grep 'model name' \| uniq -c | 计算节点：2×9554 |  |  |
| B02 | CPU压测30分钟 | stress-ng --cpu 0 --timeout 30m --metrics-brief # 另开终端监控： watch -n 2 'lscpu \| grep MHz; sensors 2>/dev/null \| head' | 无死机、无异常报错 |  |  |
| B03 | 内存容量 | free -h sudo dmidecode -t memory \| grep -E 'Size:\|Speed:' | 计算节点约1TB |  |  |
| B04 | 内存带宽stream | 编译并运行 stream（OMP多线程） export OMP_NUM_THREADS=$(nproc) ./stream 记录 Triad GB/s | 达平台理论约≥80%（记数值） |  |  |
| B05 | 内存稳定性 | sudo memtester 32G 2 # 或更长抽测 | PASS，无错误 |  |  |
| B06 | 计算节点本地盘fio | # 先确认盘符 lsblk # 顺序读（对文件或裸盘，注意别写错系统盘） fio --name=seqread --rw=read --bs=1M --size=20G --numjobs=1 --iodepth=32 --direct=1 --group_reporting --filename=/data/fiotest fio --name=seqwrite --rw=write --bs=1M --size=20G --numjobs=1 --iodepth=32 --direct=1 --group_reporting --filename=/data/fiotest fio --name=randread --rw=randread --bs=4k --size=8G --numjobs=4 --iodepth=64 --direct=1 --group_reporting --filename=/data/fiotest | SATA SSD顺序读≥500MB/s；无IO错误 |  |  |
| B07 | 存储NVMe fio | sudo nvme list # 对单盘（示例 nvme0n1，确认后执行） sudo fio --name=nvme_seq --rw=read --bs=1M --size=50G --iodepth=32 --direct=1 --filename=/dev/nvme0n1 --runtime=60 --time_based # 可选4盘并发分别记录 | 顺序读达PCIe4/5合理水平 |  |  |
| B08 | 电源冗余 | 有负载时拔单路电源线（值守） | 不停机；BMC告警 |  |  |


#### B组按节点汇总


| 节点 | CPU压测 | stream(GB/s) | 磁盘读 | 结论 |
|---|---|---|---|---|
| 管理 |  |  |  |  |
| 存储 |  |  |  |  |
| 计算1 |  |  |  |  |
| 计算2 |  |  |  |  |
| 计算3 |  |  |  |  |
| 计算4 |  |  |  |  |


### 4.3 C. GPU 与模型相关测试（重点补充）

每台计算节点执行。含：识别、显存带宽、卡间互联、单机NCCL、算力/矩阵吞吐、推理吞吐、训练吞吐、满载压力、长时间稳定性。


#### 4.3.1 C1 GPU 基础与互联


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| C01 | 8卡识别与健康 | nvidia-smi -L nvidia-smi nvidia-smi -q \| grep -iE 'Product Name\|FB Memory\|ECC\|Pending\|Xid\|Retired' | 8×A800 80GB；ECC正常；无Pending |  |  |
| C02 | 拓扑 | nvidia-smi topo -m nvidia-smi nvlink -s   # 若有NVLink | 记录NVLink/PCIe矩阵 |  |  |
| C03 | 显存带宽 | cd cuda-samples/.../bandwidthTest && make ./bandwidthTest --device=0 # 可对 device=0..7 各跑一次抽测 | HBM带宽约≥1.8TB/s（记峰值） |  |  |
| C04 | P2P/NVLink带宽 | cd .../p2pBandwidthLatencyTest && make ./p2pBandwidthLatencyTest 结合topo判断形态 | NVSwitch形态P2P很高；PCIe形态明显较低 |  |  |
| C05 | 单机8卡NCCL | cd nccl-tests/build ./all_reduce_perf -b 8 -e 8G -f 2 -g 8 # 记录末行 algbw/busbw | 按形态：NVSwitch很高；PCIe约≥45GB/s量级 |  |  |


#### 4.3.2 C2 算力、推理/训练吞吐

以下用 PyTorch；建议同一 NGC/conda 环境。先单卡再多卡。


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| C06 | 单卡GEMM算力(BF16/FP16) | python << 'PY' import torch, time assert torch.cuda.is_available() dev=0; torch.cuda.set_device(dev) n=8192; iters=50; warmup=10 a=torch.randn(n,n,device='cuda',dtype=torch.bfloat16) b=torch.randn(n,n,device='cuda',dtype=torch.bfloat16) for _ in range(warmup):   torch.mm(a,b) torch.cuda.synchronize() t0=time.time() for _ in range(iters):   torch.mm(a,b) torch.cuda.synchronize() dt=time.time()-t0 flops=2*(n**3)*iters print('TFLOPS', flops/dt/1e12) PY # dtype可改 torch.float16；对cuda:0..7循环 | A800单卡BF16/FP16接近理论较高比例（记录TFLOPS；参考约≥280更佳） |  |  |
| C07 | 单卡推理吞吐(tokens/s) | 用vLLM或transformers对小模型测速，示例vLLM： python -m vllm.entrypoints.openai.api_server --model <本地模型路径> --tensor-parallel-size 1 & # 或脚本benchmark_throughput # 也可用简单generate循环统计 tokens/s 记录：模型名、精度、batch、输入/输出长度、tokens/s | 形成单卡基线；无OOM |  |  |
| C08 | 单机8卡推理吞吐 | vLLM: --tensor-parallel-size 8 或 HuggingFace accelerate多卡 同一prompt集，记录 tokens/s 与延迟 | 明显高于单卡；扩展比记下来 |  |  |
| C09 | 单机8卡训练吞吐 | torchrun --standalone --nproc_per_node=8 train.py ... # 或官方示例/客户脚本 建议模型：Qwen2.5-7B 或 Llama-3-8B BF16 + FSDP/DDP，seq_len=2048或4096 跑≥100 step，记录 samples/s 或 tokens/s | 吞吐稳定；作为多机扩展基线 |  |  |
| C10 | 显存占用与利用率 | 训练/推理同时： nvidia-smi --query-gpu=index,utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw --format=csv -l 2 # 或 dcgmi dmon -e 1001,1002,1003,1004,1005 -d 2 | Util较高；显存均衡；无OOM |  |  |


#### 4.3.3 C3 GPU 压力与稳定性


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| C11 | gpu-burn满载1小时 | cd gpu-burn && make ./gpu_burn 3600 # 另开： nvidia-smi dmon -s pucvmet -d 5 结束后检查dmesg \| grep -i xid | 无掉卡、无Xid；温度可控(参考≤85℃)；功耗正常 |  |  |
| C12 | DCGM诊断(可选) | sudo dcgmi diag -r 3 # 或 -r 4 更深入（耗时更长） | 诊断通过或仅可接受告警 |  |  |
| C13 | 多卡同时burn | CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7 ./gpu_burn 1800 确认8卡同时高负载 | 8卡均可满载 |  |  |
| C14 | 错误复查 | nvidia-smi -q \| grep -iE 'xid\|ecc\|retired\|pending' dmesg -T \| grep -iE 'xid\|nvrm\|gpu' \| tail -50 | 无严重不可恢复错误 |  |  |


#### C组按节点汇总（含吞吐）


| 节点 | 8卡识别 | bandwidthTest | 单机NCCL | GEMM TFLOPS | 8卡训练吞吐 | gpu_burn | 结论 |
|---|---|---|---|---|---|---|---|
| 计算1 |  |  |  |  |  |  |  |
| 计算2 |  |  |  |  |  |  |  |
| 计算3 |  |  |  |  |  |  |  |
| 计算4 |  |  |  |  |  |  |  |


### 4.4 D. IB 网络


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| D01 | 链路与误码 | ibstat iblinkinfo ibcheckerrors ibswitches | Active@200G；无异常误码 |  |  |
| D02 | 写带宽 | # 服务端(节点A): ib_write_bw -d mlx5_0 --report_gbits # 客户端(节点B): ib_write_bw -d mlx5_0 --report_gbits <A_IB_IP> 双向或多对抽样 | ≥185 Gb/s |  |  |
| D03 | 读带宽 | ib_read_bw -d mlx5_0 --report_gbits <对端IP> | ≥185 Gb/s |  |  |
| D04 | 延迟 | ib_write_lat -d mlx5_0 <对端IP> | ≤2.0 μs（记实测） |  |  |
| D05 | GDR可选 | ib_write_bw -d mlx5_0 --use_cuda=0 --report_gbits (需环境支持GPU Direct RDMA) | GDR可用则带宽接近普通RDMA |  |  |
| D06 | 管理网iperf3 | # 服务端: iperf3 -s # 客户端: iperf3 -c <ip> -t 30 -P 4 | 千兆≥940Mbps |  |  |


### 4.5 E. 跨机 NCCL

环境变量示例（按实际网卡名修改）：

export NCCL_IB_HCA=mlx5_0

export NCCL_SOCKET_IFNAME=eth0   # 管理网网卡名

export NCCL_DEBUG=INFO

export NCCL_IB_DISABLE=0


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| E01 | 2机16卡all_reduce | # 示例mpirun（hostfile含2节点，每节点1进程起nccl-tests且-g 8，或每卡1进程共16） mpirun -np 2 -H node1:1,node2:1 -bind-to none -map-by slot \   -x NCCL_IB_HCA -x NCCL_DEBUG -x LD_LIBRARY_PATH \   ./all_reduce_perf -b 8 -e 8G -f 2 -g 8 # 也可每节点torchrun方式调用测试程序 记录busbw | 成功；busbw达可用区间(单口200G下常见≥20GB/s量级) |  |  |
| E02 | 4机32卡all_reduce | 扩展到4节点，同样all_reduce_perf 多跑2～3次取平均busbw | 稳定；无NCCL timeout |  |  |
| E03 | all_gather抽测 | ./all_gather_perf -b 8 -e 8G -f 2 -g 8（多机同E02） | 与all_reduce同量级、无异常 |  |  |
| E04 | 日志确认走IB | NCCL_DEBUG=INFO日志中应出现IB/NET相关，而非仅Socket | 确认走IB |  |  |


### 4.6 F. 存储共享


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| F01 | 挂载 | 按方案mount NFS/并行FS df -h \| grep <挂载点> touch <挂载点>/w_from_$(hostname) | 4计算节点均可读写 |  |  |
| F02 | 并发读写 | 4节点同时： fio --name=share --directory=<挂载点> --rw=read --bs=1M --size=10G --numjobs=4 --iodepth=16 --direct=1 | 无IO错误；记聚合带宽 |  |  |


### 4.7 G. 调度系统（简要）


| 编号 | 测试项 | 方法与步骤 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| G01 | 纳管 | 调度界面查看节点与GPU数 | 显示32卡 |  |  |
| G02 | 作业闭环 | 提交多机GPU作业→运行→结束→释放 | 无资源泄漏 |  |  |
| G03 | 监控 | 查看利用率/温度/功耗/IB流量 | 可刷新 |  |  |


### 4.8 H. 分布式训练与模型吞吐（必做）

建议公开权重：Qwen2.5-7B 或 Llama-3-8B；精度 BF16；并行 FSDP 或 DeepSpeed ZeRO；开启 flash-attention（若环境支持）。


| 编号 | 测试项 | 方法与具体命令 | 预期 | 实测 | 结论 |
|---|---|---|---|---|---|
| H01 | 环境一致 | 各节点：python -c 'import torch;print(torch.__version__, torch.version.cuda, torch.cuda.device_count())' 核对NCCL: python -c 'import torch;print(torch.cuda.nccl.version())' | 版本一致；每机可见8卡 |  |  |
| H02 | 单机8卡训练基线 | torchrun --standalone --nnodes=1 --nproc_per_node=8 \   train.py --model <path> --bf16 --seq-len 4096 --max-steps 100 记录：tokens/s或samples/s、耗时、峰值显存 | 吞吐稳定可复现 |  |  |
| H03 | 4机32卡训练 | torchrun --nnodes=4 --nproc_per_node=8 \   --rdzv_backend=c10d --rdzv_endpoint=<master管理IP>:29500 \   train.py --model <path> --bf16 --seq-len 4096 --max-steps 200 # DeepSpeed示例： # deepspeed --num_gpus 8 --num_nodes 4 --hostfile hostfile train.py --deepspeed ds_config.json 监控NCCL超时与loss | 正常启动；loss下降；无timeout |  |  |
| H04 | 扩展效率 | 效率= 32卡吞吐/(8卡吞吐×4)×100% 填写两侧吞吐与效率 | 建议≥75% |  |  |
| H05 | Checkpoint | 训练中保存ckpt；kill后从ckpt恢复再跑若干step | 可恢复继续 |  |  |
| H06 | 稳定性(建议≥2～4小时) | 32卡连续训练2～4小时 nvidia-smi -l 10 重定向日志 记录是否中断、Xid、掉卡 | 无中断、无掉卡 |  |  |
| H07 | 多机推理吞吐(可选) | vLLM多机或tensor parallel跨机（若客户需要） 记录tokens/s | 按需验收 |  |  |


#### 训练/吞吐记录表


| 项 | 填写 |
|---|---|
| 模型与精度 |  |
| 框架(FSDP/DeepSpeed等) |  |
| 启动命令 |  |
| 8卡吞吐 |  |
| 32卡吞吐 |  |
| 扩展效率% |  |
| 单卡GEMM TFLOPS(抽样) |  |
| 推理tokens/s(若测) |  |
| gpu_burn结果 |  |
| 异常与处理 |  |
| 结论 |  |


## 5. 验收标准汇总


| 项目 | 通过标准 |
|---|---|
| 配置 | 与订单一致 |
| CPU/磁盘 | 压测无故障；fio达预期量级 |
| GPU基础 | 每机8卡齐；带宽/NCCL单机达标；burn无Xid掉卡 |
| GPU吞吐 | 单卡GEMM与8卡训练吞吐有记录且稳定 |
| IB | 点到点≥185Gb/s；延迟记录合格 |
| 跨机NCCL | 32卡all_reduce成功稳定 |
| 分布式训练 | 32卡跑通；扩展效率建议≥75%；稳定性抽测通过 |


## 6. 命令速查附录

【CPU】 lscpu；stress-ng --cpu 0 --timeout 30m --metrics-brief

【内存】 free -h；./stream；sudo memtester 32G 2

【磁盘】 fio --name=seqread --rw=read --bs=1M --size=20G --iodepth=32 --direct=1 --filename=/data/fiotest

【GPU】 nvidia-smi -L；nvidia-smi topo -m；./bandwidthTest；./p2pBandwidthLatencyTest；./gpu_burn 3600

【监控】 nvidia-smi --query-gpu=index,utilization.gpu,memory.used,temperature.gpu,power.draw --format=csv -l 2

【IB】 ibstat；ib_write_bw -d mlx5_0 --report_gbits <ip>；ib_write_lat -d mlx5_0 <ip>

【NCCL单机】 ./all_reduce_perf -b 8 -e 8G -f 2 -g 8

【训练】 torchrun --nnodes=4 --nproc_per_node=8 --rdzv_backend=c10d --rdzv_endpoint=<master>:29500 train.py


## 7. 验收签字


| 验收项 | 结论 | 说明 |
|---|---|---|
| 配置核对 |  |  |
| CPU/内存/磁盘 |  |  |
| GPU基础/压力/吞吐 |  |  |
| IB / NCCL |  |  |
| 分布式训练 |  |  |
| 总体结论 |  |  |

客户签字：____________ 日期：________　　实施签字：____________ 日期：________
