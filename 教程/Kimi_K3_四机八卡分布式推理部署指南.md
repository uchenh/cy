# Kimi_K3_四机八卡分布式推理部署指南

Kimi K3

四机八卡分布式推理部署指南

4 × (8 × H200 141GB) · 共 32 卡 · Ubuntu 24.04 LTS · vLLM + Ray

适用对象：负责 GPU 集群运维与推理服务上线的工程师

配套脚本集：kimi-k3-deploy/


## 目录


## 一、文档说明

本文档描述如何在 4 台各搭载 8 张 NVIDIA H200（单卡 141GB）的服务器上，把 2.8 万亿参数级 MoE 大模型 Kimi K3 部署为一个可对外提供服务的推理实例。

文档配套一套可直接执行的 Shell 脚本（目录 kimi-k3-deploy/）。运维人员按本文的步骤顺序执行脚本即可完成部署，无需手工拼装启动命令。


### 1.1 阅读顺序建议

先读第二章，确认你的硬件确实能满足这个模型的显存要求——这一章能回答“到底跑不跑得起来”。

再读第三章，确认采用的部署拓扑与并行策略。

然后按第五章的分步操作执行，每一步都有对应的脚本与自检方法。

第六章参数速查表、第七章故障排查表供部署过程中随时查阅。


## 二、可行性结论（先看这一章）


### 2.1 模型规格


| 项目 | 数值 |
|---|---|
| 模型名称 / 仓库 ID | Kimi K3（moonshotai/Kimi-K3） |
| 架构 | Mixture-of-Experts（MoE），原生多模态（文本 + 图像） |
| 总参数 / 激活参数 | 2.8T / 104B |
| 专家配置 | 896 个专家，每 token 激活 16 个，另有 2 个共享专家 |
| 层数 | 93 层（Dense 层 1 层） |
| 注意力结构 | 69 层 KDA（Kimi Delta Attention）+ 24 层 Gated MLA，按 3:1 交替 |
| 上下文长度 | 1,048,576（1M token） |
| 量化方式 | 原生 MXFP4 权重 / MXFP8 激活（自 SFT 阶段起做量化感知训练） |
| 权重分片 | 96 个 safetensors 分片 |
| 权重体积 | 1,560,936,091,448 字节 ≈ 1560.9 GB（约 1453.7 GiB） |
| 官方最低显存要求 | 1680 GB |
| 许可 | Kimi K3 License（MIT 风格，可商用） |

为什么权重比理论值大

MXFP4 并非全部模块都做了 4bit。config.json 的 ignore 列表显示，self_attn（全部注意力层）、shared_experts、稠密 MLP 投影、lm_head、vision_tower、mm_projector 均保持高精度不参与量化，这使实际体积比“2.8T × 0.5 byte/param = 1400GB”高出约 160GB。


### 2.2 显存门槛测算

这是本方案最关键的结论：单机 8 卡装不下这个模型。


| 配置 | 总显存 | 能否承载 1560.9GB 权重 | 结论 |
|---|---|---|---|
| 1 台 × 8 卡（1 个节点） | 1128 GB | 否 | 差 433GB，单机方案不成立 |
| 2 台 × 8 卡（2 个节点） | 2256 GB | 是 | H200 上的最小可用单元 |
| 4 台 × 8 卡（4 个节点） | 4512 GB | 是 | 本方案采用 |

4 台机器合计 4512GB，扣除 1560.9GB 权重后仍余约 2.9TB 用于 KV 缓存与激活，配置相当宽裕。因此“4 台八卡机”这个规模对本模型是合理的，但前提是必须把 32 张卡聚合成一个实例。

由此推出的第一个关键结论

“每台机器跑一个副本”在本硬件上不可行——8 卡只有 1128GB，连权重都放不下。H200 上每副本至少需要 16 卡。这也是为什么本方案的主推形态是 32 卡单实例跨 4 机。


### 2.3 三条硬约束


#### 约束一：宿主机驱动必须 ≥ r580

官方镜像 vllm/vllm-openai:kimi-k3 只提供 CUDA 13（cu130）构建，没有 -cu129 标签，启用 K3 的 wheel 也不在 cu129 nightly 索引中。宿主机若为 CUDA 12.9（r575）必须先升级驱动，否则容器起不来。


#### 约束二：vLLM 的 K3 配方目前是 Pre-release

官方配方标注 nightly_required: true、难度 hard，参数与镜像行为可能随版本变化。首次部署成功后，必须把镜像 digest 固化到 env.sh 中，否则后续 tag 漂移会导致行为不一致。


#### 约束三：跨机只能用 RDMA，且后端选择受限

官方 Notes 明确：RDMA 跨机用 deepep_v2；而 deep_gemm_mega_moe 与跨节点 RDMA 不兼容。本集群是 IB NDR 400G，因此本文脚本在配置校验阶段就会直接拦下填错的后端名称。


## 三、集群环境与拓扑选型


### 3.1 环境清单


| 层面 | 配置 | 说明 |
|---|---|---|
| 操作系统 | Ubuntu 24.04 LTS (x86_64) | 可直接使用，无需更换发行版 |
| GPU | 4 × (8 × NVIDIA H200 141GB) | H200 在 vLLM 官方配方中为 verified 硬件 |
| GPU 机内互联 | NVLink / NVSwitch | TP=8 严格留在机内，充分利用机内高带宽 |
| 节点间网络 | InfiniBand NDR 400G | 跨机 EP all2all 通道，需 GPUDirect RDMA |
| 宿主驱动 | NVIDIA r580+ | 硬门槛，见 2.3 |
| 容器运行时 | Docker + nvidia-container-toolkit | 统一四节点环境，规避依赖漂移 |
| 推理引擎 | vLLM ≥ 0.27.0（K3 分支） | 镜像内已包含 |
| 分布式运行时 | Ray | 跨机调度，多节点 vLLM 默认即 Ray |
| 权重来源 | 魔搭 ModelScope | moonshotai/Kimi-K3 |


### 3.2 主推拓扑 A：32 卡单实例跨 4 机


| 项目 | 取值 | 理由 |
|---|---|---|
| 张量并行 TP | 8（机内） | TP 对带宽和时延极敏感，必须留在机内走 NVLink，绝不跨机 |
| 数据并行 DP | 4（每节点 1 个 rank） | 每个 DP rank 需要 8 张卡，正好对应一台机器 |
| 专家并行 EP | 32（= TP × DP） | 896 个专家的权重全集群分片，是 MoE 的正确解法 |
| 跨机 all2all | deepep_v2 | 官方指定的 RDMA 跨机后端 |
| MoE 计算后端 | marlin | H200（Hopper）专属要求，不得沿用 Blackwell 取值 |
| H200 覆写参数 | no-enable-flashinfer-autotune disable-custom-all-reduce | 缺省会导致启动失败或性能异常 |
| 启动方式 | LAUNCH_MODE=ray | 仅 node0 执行一条命令，Ray 自动分发 4 个 DP rank |

为什么跨机用 EP 而不是 PP

PP 是按层切分，每个节点仍要保存自己那部分层的全部专家权重，并不能解决显存容量问题；而 896 专家的大规模 MoE 天生适合 EP——把专家分散到 32 张卡上。因此本方案跨机只承担 EP 的 all2all 流量，TP 的 allreduce 严格留在机内。


### 3.3 备选拓扑


| 拓扑 | 形态 | 适用场景 | 切换方式 |
|---|---|---|---|
| A（默认） | 32 卡单实例跨 4 机 | 最大化单实例容量，支撑 1M 长上下文与高并发 | TOPOLOGY=single（默认） |
| B | 2 副本 × 16 卡 + 网关 | 并发优先、需要冗余；任一副本故障可降级 | TOPOLOGY=replica，配 nginx/kimi-k3.conf |
| C | Prefill / Decode 分离 | 官方生产方案，两侧用不同并行策略 | 复杂度最高，建议有压测数据后再演进 |
| 兜底 | 无 Ray，走 --nnodes/--node-rank | Ray 出问题时的降级路径，或不想引入 Ray | LAUNCH_MODE=mp |

另有一个框架备选：SGLang 的官方 cookbook 中，H200 恰好有 4 节点 × 8 卡的 TP32/EP32 配方，可读性比 vLLM 侧更好。但官方将其标注为“Final Verification In Progress”，因此本文以 vLLM 为主线，SGLang 仅作为交叉验证与兜底。


### 3.4 关键决策与取舍


| 决策点 | 选择 | 理由 / 代价 |
|---|---|---|
| KV 缓存分布式存储 | 不采用 | Mooncake 的分布式/集中式 KV 缓存在 H200 等全部目标硬件上均标记为 unsupported |
| 镜像 tag 还是 digest | 固定 digest | 配方为 Pre-release，参数与镜像会变，复现性优先 |
| 是否引入 Ray | 默认引入，保留降级路径 | Ray 单条命令拉起全部 rank；但它是额外故障面，故保留 mp 路径 |
| 上下文长度起步值 | 128K（131072） | 不要一上来就开 1M，按 128K → 256K → 512K → 1M 逐级压测 |
| FP8 KV 缓存 | 默认关闭 | 开启可让 KV 减半，但需同时配置 attention-config，且影响精度，按压测结果决定 |


## 四、部署前准备


### 4.1 必改配置项

所有可变参数都收敛在 kimi-k3-deploy/env.sh 这一个文件里，其余脚本不含硬编码参数。部署前只需改这一个文件，且除 NODE_RANK 外，其余内容在四台机器上必须完全一致。


| 变量 | 示例值 | 约束 |
|---|---|---|
| NODE_IPS | ("10.0.0.11" "10.0.0.12" "10.0.0.13" "10.0.0.14") | 数组顺序即 NODE_RANK，必须是私有网段地址 |
| MGMT_IFNAME | eno1 | 管理网卡名，Ray/Gloo 控制面使用 |
| IB_HCA | mlx5 | 必须与 ibstat 实际输出一致 |
| IB_IFNAME | ibp0s0 | RDMA 接口名 |
| MODEL_LOCAL_PATH | /data/models/Kimi-K3 | 四节点必须完全一致 |
| VLLM_IMAGE_DIGEST | sha256:... | 首次部署后固化 |
| TP_SIZE / DP_SIZE | 8 / 4 | TP 必须等于单机卡数，DP 必须等于节点数 |
| ALL2ALL_BACKEND | deepep_v2 | 跨机 RDMA 唯一正确取值 |
| MOE_BACKEND | marlin | H200 必须为 marlin |
| MAX_MODEL_LEN | 131072 | 起步 128K，压测后再上调 |


### 4.2 首次使用：转换行尾并做语法自检

脚本在 Windows 上编辑过，行尾为 CRLF 时 Linux 会报 “/usr/bin/env: 'bash\r': No such file or directory”。首次使用前必须先转换：

cd kimi-k3-deploy

find . -name '*.sh' -exec sed -i 's/\r$//' {} +

find . -name '*.sh' -exec chmod +x {} +

# 语法自检，所有脚本都应无输出

for f in $(find . -name '*.sh'); do bash -n "$f" || echo "SYNTAX FAIL: $f"; done


### 4.3 前置条件核对

四台机器均已安装 Ubuntu 24.04 LTS，内网互通，SSH 免密登录已配置

每台机器 8 张 H200 141GB，机内 NVLink/NVSwitch 正常

节点间 IB NDR 400G，ibstat 显示 State=Active、Rate=400

NVIDIA 驱动 ≥ r580

每个节点预留至少 1900GB 磁盘（权重 1561GB + 余量）

四台机器上脚本目录的绝对路径一致（否则 06_stop.sh 的远端调用会失败）

所有脚本需 root 权限执行


## 五、分步部署操作


### 5.1 步骤一：环境自检（只读）

在下载 1.56TB 权重之前先跑这一步，把代价最高的失败提前拦掉。每台机器都要执行，NODE_RANK 分别填 0 到 3。

NODE_RANK=0 ./00_check_env.sh

NODE_RANK=1 ./00_check_env.sh

NODE_RANK=2 ./00_check_env.sh

NODE_RANK=3 ./00_check_env.sh

脚本逐项输出 PASS / WARN / FAIL 汇总表，覆盖 10 类检查：


| # | 检查项 | 判定标准 |
|---|---|---|
| 1 | 操作系统与架构 | Ubuntu 24.04 / x86_64 |
| 2 | GPU 型号、数量、单卡显存、集群总显存、机内 NVLink | 8 张 H200；总显存 ≥ 1680GB |
| 3 | NVIDIA 驱动版本 | 主版本号 ≥ 580 |
| 4 | IB 设备、端口状态、链路速率、nvidia_peermem、memlock | State=Active，Rate ≥ 400 |
| 5 | Docker 与 nvidia-container-toolkit | 均已安装且 daemon 可访问 |
| 6 | 推理镜像是否存在、digest 是否与配置一致 | 存在且 digest 匹配 |
| 7 | 权重分片数与核心文件、量化配置 | 96/96 分片，format=mxfp4-pack-quantized |
| 8 | 磁盘可用空间 | ≥ 2000GB |
| 9 | 节点连通性、SSH 免密、关键端口占用 | 其余节点全部可达 |
| 10 | memlock 持久化、网络缓冲、NTP 时间同步、nvidia 内核模块 | 无 FAIL |

判定规则

只要出现任何一项 FAIL，脚本会以非零退出码结束，此时不要继续后续步骤。WARN 不影响继续，但建议逐条确认原因。


### 5.2 步骤二：宿主环境配置

每台机器分别执行。脚本会安装 Docker 与 nvidia-container-toolkit、RDMA 用户态工具、加载 nvidia_peermem、写入内核参数，并拉取推理镜像。所有改动都收敛在独立文件中，便于回滚。

sudo NODE_RANK=0 ./01_setup_host.sh --yes

# node1 / node2 / node3 分别替换 NODE_RANK 后执行

脚本写入的配置文件：

/etc/security/limits.d/99-kimi-k3.conf —— memlock 与 nofile 限制

/etc/sysctl.d/99-kimi-k3.conf —— 网络缓冲、BDP、ip_forward、overcommit

/etc/modules-load.d/kimi-k3.conf —— nvidia_peermem 开机加载

/etc/docker/daemon.json —— 注册 nvidia runtime（会先备份原文件）

容易漏掉的一点

memlock 的 limits 修改对已登录的会话不生效，需要重新登录。不过容器启动时已使用 --ulimit memlock=-1 兜底，所以不会阻塞部署，但宿主机上直接跑 RDMA 工具时仍可能受限。


### 5.3 步骤三：权重获取与分发

从魔搭（ModelScope）下载。权重总量约 1560.9GB，按 10Gbps 带宽估算约 21 分钟，按 1Gbps 估算约 3.5 小时。支持断点续传，中断后重跑即可。

./02_download_weights.sh precheck      # 秒级：先确认量化格式与分片数

./02_download_weights.sh all           # 全量下载 96 个分片 + 校验

./02_download_weights.sh distribute    # 分发到其余 3 个节点（可选）

三种获取策略，按你的网络条件选择：


| 策略 | 做法 | 适用条件 |
|---|---|---|
| 单点下载 + 分发 | 在一台机器下载后 rsync 到其余三台 | 节点间带宽充足，或下载出口带宽受限 |
| 四节点并行下载 | 每台机器各自执行 download | 下载出口带宽充足，可省去节点间传输 |
| 共享文件系统 | 权重放在 NAS/分布式 FS，四节点挂载到同一路径 | 已有共享存储，且带宽满足加载需求 |

路径一致性是硬要求

四台机器上 MODEL_LOCAL_PATH 必须完全相同。容器按相同绝对路径挂载（只读），脚本不做路径映射。若路径不一致，引擎加载权重时会直接失败。


### 5.4 步骤四：组建 Ray 集群

本步骤会在每个节点创建统一的运行时容器（所有环境变量、RDMA 设备、权重目录一并就位），然后组建 Ray 集群。先起 node0（head），再起其余节点（worker）。

# 先 node0

NODE_RANK=0 ./03_ray_cluster.sh start

# 再 node1 / node2 / node3

NODE_RANK=1 ./03_ray_cluster.sh start

NODE_RANK=2 ./03_ray_cluster.sh start

NODE_RANK=3 ./03_ray_cluster.sh start

# 回到 node0 校验集群规模

NODE_RANK=0 ./03_ray_cluster.sh status

校验必须看到 4 个节点、32 个 GPU，否则不要继续。容器创建时的关键参数：

--gpus all --network host --ipc host —— 容器与宿主共享网络与 IPC

--privileged --device /dev/infiniband —— RDMA 设备直通

--ulimit memlock=-1 --ulimit stack=67108864 —— RDMA 锁页与大栈

-v 权重目录:同一路径:ro —— 只读挂载，避免误改 1.56TB 权重

-e VLLM_HOST_IP=<本机 IP> —— 每节点各不相同


### 5.5 步骤五：启动推理服务

LAUNCH_MODE=ray（默认）时，只需在 node0 执行一条命令，Ray 会自动把 4 个 DP rank 分发到四台机器。

# 先干跑一次，确认命令正确

NODE_RANK=0 ./04_serve_vllm.sh dry-run

# 正式启动并等待就绪

NODE_RANK=0 ./04_serve_vllm.sh start --wait

脚本实际下发的核心参数：

vllm serve /data/models/Kimi-K3 \

--tensor-parallel-size 8 \

--data-parallel-size 4 \

--data-parallel-backend=ray \

--distributed-executor-backend ray \

--enable-expert-parallel \

--all2all-backend deepep_v2 \

--moe-backend marlin \

--no-enable-flashinfer-autotune \

--disable-custom-all-reduce \

--load-format fastsafetensors \

--gpu-memory-utilization 0.95 \

--max-model-len 131072 \

--max-num-seqs 32 \

--reasoning-parser kimi_k3 \

--tool-call-parser kimi_k3 \

--enable-auto-tool-choice

首启动需要耐心

1.56TB 权重加载通常需要 10 到 40 分钟。为此必须把引擎就绪超时放宽到 3600 秒（脚本已通过环境变量 VLLM_ENGINE_READY_TIMEOUT_S 设置）。若看到超时报错，先确认这一点，而不是急着判定部署失败。

如果 LAUNCH_MODE 改为 dp 或 mp，则四台机器都要执行一次 start。mp 模式不使用专家并行，MoE 专家层会退化为 TP 分片，吞吐通常低于 ray/dp 模式，仅作兜底。

# LAUNCH_MODE=dp 或 mp 时，四台机器分别执行

NODE_RANK=0 ./04_serve_vllm.sh start --wait

NODE_RANK=1 ./04_serve_vllm.sh start --wait

NODE_RANK=2 ./04_serve_vllm.sh start --wait

NODE_RANK=3 ./04_serve_vllm.sh start --wait


### 5.6 步骤六：部署验证

NODE_RANK=0 ./05_verify.sh

脚本执行五类验证并输出结构化报告：


| 验证项 | 判定方式 | 不合格时的意义 |
|---|---|---|
| RDMA 是否真正生效 | 日志中出现 NET/IB/GDRDMA；出现 NET/Socket 则告警 | 退化为 TCP 会让跨机 EP 严重劣化 |
| KV cache 容量 | 解析日志中的 GPU KV cache size 与 Maximum concurrency | 决定并发上限，是容量规划依据 |
| 服务健康检查 | GET /health 返回 200 | 服务未就绪或已崩溃 |
| 纯文本请求 | 返回 200 且结构正确；检查是否含 reasoning_content | K3 始终开启 thinking，缺失说明解析器配置有误 |
| 图像输入请求 | 多模态代码路径返回 200 | 验证 MoonViT-V2 视觉编码器链路 |
| 流式输出 | 收到首个 SSE 数据块并记录 TTFT | SSE 不通通常是网关开启了缓冲 |
| 工具调用 | 返回 tool_calls 且结构可解析 | K3 偶发输出解析器不接受的格式，集成层必须校验+重试 |


### 5.7 步骤七：压测与容量标定

本文不给出任何推测的性能数字。请用压测脚本建立容量曲线后，再固化生产参数。

# 默认并发 1 / 4 / 8

./scripts/bench.sh --concurrency 1,4,8 --prompt-tokens 32000

# 结果落在 /var/log/kimi-k3/bench-<时间戳>.csv

逐级扩容的建议流程：

先以 MAX_MODEL_LEN=131072（128K）启动，跑一轮并发 1/4/8，记录 TTFT、输出吞吐与日志中的并发上限。

把 MAX_MODEL_LEN 依次提升到 262144、524288、1048576，每次都重启服务并重复压测。

观察 TTFT 随并发上升的拐点，那一点就是当前 KV cache 与 batch 配置下的实用并发上限。

当输出 tok/s 不再随并发增长时，说明已触达算力或专家并行通信瓶颈。

用 CSV 数据确定满足 SLA 的最大上下文长度与 max-num-seqs 组合，写入 env.sh 后固化。


## 六、参数速查表


### 6.1 启动参数


| 参数 | 取值 | 说明 |
|---|---|---|
| --tensor-parallel-size | 8 | 等于单机卡数，机内 NVLink |
| --data-parallel-size | 4 | 等于节点数，每个 DP rank 占一台机器 |
| --enable-expert-parallel | 启用 | EP 组大小自动等于 TP × DP = 32；所有节点必须一致 |
| --all2all-backend | deepep_v2 | RDMA 跨机唯一正确取值 |
| --moe-backend | marlin | H200/Hopper 专属；Blackwell 会自动选 FlashInfer MXFP4，不可照搬 |
| --no-enable-flashinfer-autotune | 启用 | H200 覆写 |
| --disable-custom-all-reduce | 启用 | H200 覆写 |
| --load-format | fastsafetensors | 显著加快 1560GB 权重的加载 |
| --gpu-memory-utilization | 0.95 | 留给 KV 缓存的显存比例 |
| --max-model-len | 131072 | 起步值，逐级压测至 1048576 |
| --max-num-seqs | 32 | 按每个 DP rank 生效 |
| --reasoning-parser | kimi_k3 | K3 始终输出 reasoning_content |
| --tool-call-parser | kimi_k3 | 需配合 --enable-auto-tool-choice |
| --kv-cache-dtype | fp8 | 可选；需同时配置 attention-config 才生效 |


### 6.2 关键环境变量


| 变量 | 值 | 作用 |
|---|---|---|
| VLLM_ENGINE_READY_TIMEOUT_S | 3600 | 必设。1560GB 权重加载远超默认超时 |
| UCX_TLS | rc,cuda_copy | 必设。否则 KV 缓存传输可能不走 RDMA |
| VLLM_RAY_DP_PACK_STRATEGY | span | 单个 DP 组跨多节点时必须设置 |
| NCCL_IB_HCA | mlx5 | 指定使用的 IB 设备 |
| NCCL_SOCKET_IFNAME | ibp0s0 | 指定 NCCL 使用的网卡 |
| GLOO_SOCKET_IFNAME | ibp0s0 | 指定 Gloo 使用的网卡 |
| NCCL_CUMEM_ENABLE | 1 | 跨机通信优化 |
| NCCL_MNNVL_ENABLE | 1 | 跨机通信优化 |
| NCCL_NVLS_ENABLE | 0 | 本集群非 GB200/GB300 NVL 场景 |
| VLLM_ENABLE_K3_LATENT_MOE_TAIL_FUSION | 1 | K3 专属算子融合优化 |
| VLLM_ALLREDUCE_USE_FLASHINFER | 1 | allreduce 走 FlashInfer |
| VLLM_USE_V2_MODEL_RUNNER | 1 | vLLM 新 Model Runner，官方称完全支持本模型 |
| VLLM_USE_RUST_FRONTEND | 1 | Rust 前端，官方称完全支持本模型 |
| VLLM_SSM_CONV_STATE_LAYOUT | DS | KDA/SSM 状态布局 |
| NCCL_DMABUF_ENABLE | 0 | 仅在遇到 mlx5dmabuf 524 错误时添加，作为回退 |
| NCCL_DEBUG | TRACE | 仅诊断时开启，用于确认 RDMA 是否生效；生产请用 WARN |


### 6.3 容量口径易错点

两个参数的作用域完全不同

--max-num-seqs 是按“每个 DP rank”生效的；而 --max-num-queued-reqs 和 --max-num-queued-tokens 是按“整个 server”生效的。做容量规划时如果混用，会得到完全错误的并发估算。建议把队列上限设为 ≈ data-parallel-size × max-num-seqs 再加上期望的排队深度。


## 七、故障排查速查表


| 现象 | 可能原因 | 处理方式 |
|---|---|---|
| /usr/bin/env: 'bash\r' | 行尾为 CRLF | 执行 sed -i 's/\r$//' 转换所有 .sh 文件 |
| 环境自检报驱动版本 FAIL | 驱动低于 r580 | 镜像仅有 cu130 构建，需升级驱动 |
| 容器内 nvidia-smi 失败 | toolkit 未安装或未配置 | 重跑 ./01_setup_host.sh |
| 03 脚本 status 显示 GPU 少于 32 | 某节点容器未起、GPU 被占用、或防火墙挡了 Ray 端口 | 逐节点查看 ray 日志与容器状态 |
| 启动后 10 秒内进程退出 | 参数不被当前 vLLM 版本识别、显存不足、权重路径不可读 | 看日志末尾 50 行，脚本已自动打印 |
| 启动长时间无响应后超时 | 权重加载耗时 10~40 分钟属正常 | 确认 ENGINE_READY_TIMEOUT_S=3600 已生效 |
| 日志出现 NET/Socket | RDMA 未生效 | 检查 IB_HCA/IB_IFNAME、/dev/infiniband 挂载、UCX_TLS |
| NCCL unhandled system error（524） | 内核/驱动缺少 mlx5 dmabuf 支持 | 加 NCCL_DMABUF_ENABLE=0，并确认已加载 nvidia_peermem |
| 禁用 flashinfer autotune 后仍异常 | H200 覆写参数未全部生效 | 确认三项 H200 覆写都在启动命令里 |
| 工具调用偶发解析失败 | K3 会输出自身 parser 不接受的格式（已知问题） | 集成层必须做 schema 校验 + 重试 |
| 并发上不去 | 容量口径算错，或已触达通信瓶颈 | 核对 max-num-seqs 与 max-num-queued-* 的作用域；跑 bench.sh 找拐点 |
| OOM 但显存看起来够 | gpu-memory-utilization 过高，或激活峰值超预期 | 下调到 0.90 并降低 max-num-batched-tokens |

日志位置（默认 /var/log/kimi-k3/）：

vllm-<节点名>-current.log —— 服务启动与运行日志（软链，指向最新）

ray/ —— Ray head 与 worker 的启动日志

bench-<时间戳>.csv —— 压测结果

systemd-ray.log / systemd-serve.log —— 使用 systemd 托管时的输出


## 八、日常运维


### 8.1 常用命令

NODE_RANK=0 ./04_serve_vllm.sh status      # 服务状态 + KV cache 指标

NODE_RANK=0 ./04_serve_vllm.sh logs        # 跟踪日志

NODE_RANK=0 ./04_serve_vllm.sh stop        # 只停推理服务

NODE_RANK=0 ./03_ray_cluster.sh status     # Ray 集群规模

NODE_RANK=0 ./06_stop.sh                   # 停止服务与 Ray（保留容器）

NODE_RANK=0 ./06_stop.sh --clean           # 额外移除容器与临时目录


### 8.2 开机自启（可选）

提供两个 systemd 单元，如需托管请这样部署：

sudo cp systemd/kimi-k3-ray.service   /etc/systemd/system/

sudo cp systemd/kimi-k3-serve.service /etc/systemd/system/

echo 'NODE_RANK=0' | sudo tee /etc/default/kimi-k3   # 每台机器填自己的编号

sudo systemctl daemon-reload

sudo systemctl enable --now kimi-k3-ray.service kimi-k3-serve.service

systemd 超时必须放宽

kimi-k3-serve.service 的 TimeoutStartSec 已设为 7200 秒。如果沿用默认的 90 秒，systemd 会在权重还没加载完时就把服务杀掉，表现为“启动失败但日志里没有任何错误”。


### 8.3 客户端调用

服务对外暴露 OpenAI 兼容接口。注意 K3 的几个调用侧特性：

from openai import OpenAI

client = OpenAI(api_key="EMPTY", base_url="http://10.0.0.11:8000/v1",

timeout=3600)   # 超时需放大

resp = client.chat.completions.create(

model="kimi-k3",

messages=[{"role": "user", "content": "你好"}],

reasoning_effort="max",     # low / high / max，默认 max

max_tokens=2048,

)

print(resp.choices[0].message.reasoning)

K3 始终开启 thinking，响应中总会带 reasoning_content

采用 preserved thinking history：多轮对话与工具调用时，必须把上一次返回的完整 assistant 消息（含 reasoning_content 与 tool_calls）原样回传，而不是只回传 content

reasoning_effort 默认 max，输出 token 量偏大，需按延迟预算规划超时与并发


## 九、风险提示与不确定项


### 9.1 必须知晓的风险


| # | 风险 | 影响与应对 |
|---|---|---|
| 1 | 官方配方为 Pre-release，nightly 必需 | 镜像与参数会变。生产必须固定 image digest，并在升级前于测试环境回归 |
| 2 | H200 虽标记 verified，但需 marlin 等覆写参数 | 遗漏会导致启动失败或性能异常，脚本已固化这三项，勿手工删改 |
| 3 | RDMA 生效性无法靠配置推断，只能实测 | 每次变更网卡或驱动后，都要用 NCCL_DEBUG=TRACE 复验日志中的 NET/IB/GDRDMA |
| 4 | 1M 上下文会显著挤压 KV cache | 不要一开始就开满，按 128K 起步逐级压测 |
| 5 | MoE 各 DP rank 每步强制同步 | 批次不均匀时实际吞吐低于理论上限，容量规划需留余量 |
| 6 | KV 缓存分布式存储不被支持 | 架构上不要按 Mooncake 分布式 KV 方向设计 |


### 9.2 无公开基准、需自行实测校准的部分

本文刻意不给出性能数字

以下三项在公开资料中均无可靠基准：32×H200 上 K3 的实际 KV cache 容量与可用并发；H200 搭配 deepep_v2 的实测吞吐；单副本 32 卡与 2×16 卡副本之间的取舍。因此本文给出的是方法、参数起点与验证手段，而不是虚构的 TPS 或 TTFT 数字。

请先用 05_verify.sh 确认功能全部通过。

再用 scripts/bench.sh 建立自己的容量曲线。

最后把实测得到的最优参数回写到 env.sh 并固化版本。


## 十、附录：脚本清单


| 文件 | 作用 | 执行范围 |
|---|---|---|
| env.sh | 唯一配置源：IP、路径、并行度、镜像、环境变量 | 修改一次 |
| lib/common.sh | 公共函数库：日志、错误处理、容器封装、配置校验 | 被引用 |
| 00_check_env.sh | 环境自检（只读）：硬件、驱动、RDMA、容器、磁盘、权重 | 四节点分别执行 |
| 01_setup_host.sh | 宿主配置：Docker、NVIDIA Toolkit、RDMA、内核参数、拉镜像 | 四节点分别执行 |
| 02_download_weights.sh | 魔搭下载 96 分片、校验、分发 | 单点或四节点 |
| 03_ray_cluster.sh | 运行时容器 + Ray 集群 + 32 卡规模校验 | 四节点分别执行 |
| 04_serve_vllm.sh | 启动推理服务（ray / dp / mp 三种模式自动分派） | ray 模式仅 node0 |
| 05_verify.sh | 部署验证：RDMA 生效性、容量指标、四类冒烟测试 | API 节点执行 |
| 06_stop.sh | 按相反顺序优雅停止服务与 Ray，可选清理容器 | node0 统一执行 |
| nginx/kimi-k3.conf | 拓扑 B 双副本的网关配置（含流式与长超时处理） | 网关节点 |
| systemd/*.service | 可选的容器与推理服务开机自启单元 | 四节点分别部署 |
| scripts/bench.sh | 轻量压测：并发梯度下的 TTFT / ITL / 吞吐采集 | API 节点执行 |
