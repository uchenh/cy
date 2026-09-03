# 软件环境完整部署操作步骤

> 适用于：Ubuntu 22.04/24.04 x86\_64 + NVIDIA GPU（H200 单机 4 卡已验证，其它卡数需调整 `tensor-parallel-size`）。
> 目标部署内容：Docker、Docker Compose、NVIDIA vLLM 26.07-py3、官方 vLLM v0.27.1、DeepSeek-V4-Flash-0731（docker-compose 运行）、rdb、Redis、Python、Node.js。
> 全文命令默认以 `root` 执行；若用普通用户，命令前加 `sudo`。

***

## 〇、准备阶段

### 1. 确认硬件与系统

```bash
# 系统信息
cat /etc/os-release | grep -E "PRETTY_NAME|VERSION_CODENAME"
# 内存
free -h
# 磁盘（模型 159G + Docker 镜像 60G+，建议至少 400G 可用）
df -h /data   # 数据放 /data（若 /data 不存在，见下文挂载步骤）
# GPU 型号与数量
nvidia-smi
# 驱动是否就绪
nvidia-smi | head -3
```

> DeepSeek-V4-Flash-0731 需 **4 卡 TP=4** 实测通过。显存按 H200（143771 MiB）计算；若用 48G 卡（如 L40S/A6000），8 卡方可承载，需相应调整 TP 与 gpu-memory-utilization。

### 2. 准备磁盘目录

```bash
mkdir -p /data/models /data/deploy /data/docker
```

> 若 /data 不在根分区而是独立盘/软RAID，先 mount 好再建目录。

### 3. 网络检查（代理可选）

国内机器拉取 Docker Hub / modelscope 可能偏慢，必要时挂代理：

```bash
export http_proxy=http://<proxy_host>:<port>
export https_proxy=http://<proxy_host>:<port>
```

> 文档配套脚本将通过 SSH 免交互执行（`remote.py`），见文末「附：remote.py 用法」。

***

## 一、安装 Docker + Docker Compose

执行 `install_docker.sh`（阿里云源 + 国内镜像加速 + compose 插件）：

```bash
# 逐条执行（与脚本等价）
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y ca-certificates curl gnupg lsb-release

# 添加 Docker 阿里云 GPG 与源
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
ARCH=$(dpkg --print-architecture)
CODENAME=$(. /etc/os-release && echo "$VERSION_CODENAME")
echo "deb [arch=${ARCH} signed-by=/etc/apt/keyrings/docker.asc] https://mirrors.aliyun.com/docker-ce/linux/ubuntu ${CODENAME} stable" > /etc/apt/sources.list.d/docker.list
apt-get update -y
apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 启动 + 配置国内镜像加速（关键，否则拉 vLLM 可能失败/极慢）
systemctl enable --now docker
mkdir -p /etc/docker
cat > /etc/docker/daemon.json <<'EOF'
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com",
    "https://docker.nju.edu.cn",
    "https://docker.mirrors.ustc.edu.cn"
  ],
  "data-root": "/data/docker",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {"max-size": "100m", "max-file": "3"}
}
EOF
systemctl daemon-reload
systemctl restart docker

# 验证
docker --version
docker compose version
```

**验证通过标志**：`docker --version` 与 `docker compose version` 正常输出；`docker info` 显示 `Server Version`。

***

## 二、安装基础依赖（Python / Node.js / Redis / rdb）

执行 `install_deps.sh`：

```bash
# ==== [A] Python ====
export DEBIAN_FRONTEND=noninteractive
apt-get update -y
apt-get install -y python3 python3-pip python3-venv python3-dev build-essential
ln -sf /usr/bin/python3 /usr/local/bin/python
ln -sf /usr/bin/pip3 /usr/local/bin/pip
python3 --version
python3 -m pip install --upgrade pip

# ==== [B] Node.js 20 LTS ====
apt-get install -y curl ca-certificates gnupg
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt-get install -y nodejs
node -v && npm -v

# ==== [C] Redis ====
apt-get install -y redis-server
systemctl enable --now redis-server
redis-cli ping   # 期望 PONG
```

rdb（redis-rdb-tools）源码安装，执行 `install_rdb3.sh`：

```bash
cd /tmp
rm -rf redis-rdb-tools
git clone --depth 1 https://github.com/sripathikrishnan/redis-rdb-tools
cd /tmp/redis-rdb-tools
python3 setup.py install 2>&1 | tail -6 || pip3 install --break-system-packages . 2>&1 | tail -6
hash -r
which rdb && rdb --version   # 期望 usage: rdb [options] /path/to/dump.rdb
```

**版本参考（本次实测）**：Python 3.12.3 / Node v20.20.2 / npm 10.8.2 / Redis 7.0.15 / rdb 0.1.15。

***

## 三、安装 NVIDIA Container Toolkit（GPU 运行时）

执行 `install_toolkit.sh`：

```bash
# 添加 NVIDIA Container Toolkit 官方源
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg 2>/dev/null \
  || curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | gpg --dearmor --yes -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' > /etc/apt/sources.list.d/nvidia-container-toolkit.list
apt-get update -y
apt-get install -y nvidia-container-toolkit

# 配置 docker runtime 并重启
nvidia-ctk runtime configure --runtime=docker
systemctl restart docker
sleep 3

# 验证 GPU 可见
docker run --rm --gpus all nvidia/cuda:12.0.0-base-ubuntu22.04 nvidia-smi | head -15
```

**验证通过标志**：容器内 `nvidia-smi` 能列出全部 GPU。

***

## 四、下载模型 DeepSeek-V4-Flash-0731

### 4.1 准备工具 venv（含 modelscope）

下载工具用隔离 venv，避免污染系统 python：

```bash
python3 -m venv /opt/modelscp
/opt/modelscp/bin/pip install -U modelscope 2>&1 | tail -3
export PATH=/opt/modelscp/bin:$PATH
python3 -c "from modelscope import snapshot_download; print('modelscope ok')"
```

### 4.2 稳健断点续传下载

模型 48 分片 / 约 159 GiB。直接跑 `download_model_robust.sh`（短超时续传，卡住自愈）：

```bash
# 脚本核心逻辑（每轮最多 850s，失败即重试）
export PATH=/opt/modelscp/bin:$PATH
cd /data/models
MAX=48
for round in $(seq 1 60); do
  DONE=$(find /data/models/DeepSeek-V4-Flash-0731 -maxdepth 1 -name '*.safetensors' ! -name '*.incomplete' 2>/dev/null | wc -l)
  echo ">>> round=$round 完成分片=$DONE/$MAX"
  [ "$DONE" -ge "$MAX" ] && { echo "ALL_SHARDS_DOWNLOADED"; break; }
  timeout 850 python3 -c "
import os
os.environ.setdefault('MODELSCOPE_CACHE','/data/models/modelscope')
from modelscope import snapshot_download
snapshot_download('deepseek-ai/DeepSeek-V4-Flash-0731',
    local_dir='/data/models/DeepSeek-V4-Flash-0731')
print('round_ok')
" || { echo "round timeout/failure, retry"; sleep 2; }
done
```

> 重要经验：
>
> * 务必用 `/opt/modelscp/bin/python3`（系统 python3 无 modelscope，续传会报 `ImportError`）
>
> * 用 `timeout 850` 包住，连接挂起时卡住会被超时打断，再重试即可续传
>
> * 若想后台运行：`nohup bash download_model_robust.sh > /tmp/dl.log 2>&1 &`
>
> * 长时间下载建议脱离 SSH 会话（nohup 或 tmux），避免会话中断导致下载失去守护

**下载完成标志**：输出 `ALL_SHARDS_DOWNLOADED`，或 `find /data/models/DeepSeek-V4-Flash-0731 -maxdepth 1 -name '*.safetensors' | wc -l` >= 48。

***

## 五、拉取两个 vLLM 镜像

### 5.1 官方 vLLM v0.27.1（DeepSeek 实际推理引擎）

执行 `pull_of_vllm.sh`：

```bash
docker pull vllm/vllm-openai:v0.27.1
docker images vllm/vllm-openai:v0.27.1   # 期望 30.8GB 就位
```

### 5.2 NVIDIA NGC vLLM 26.07-py3（备选）

执行 `pull_nv_vllm.sh`：

```bash
docker pull nvcr.io/nvidia/vllm:26.07-py3
docker images nvcr.io/nvidia/vllm:26.07-py3   # 期望 33.3GB 就位
```

> 若 NGC 极慢/失败，先检查 daemon.json 镜像加速是否生效；必要时代理后重试。

***

## 六、编写 docker-compose.yml 并部署

### 6.1 创建部署目录与配置

```bash
mkdir -p /data/deploy
```

在 `/data/deploy/docker-compose.yml` 填入如下配置（`install` 已有脚本 `prepare_deploy.sh` 可上传）：

```yaml
services:
  deepseek-v4-flash:
    image: vllm/vllm-openai:v0.27.1        # 官方 vLLM 镜像（4 GPU TP）
    container_name: deepseek-v4-flash-0731
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
      - HF_HOME=/root/.cache/huggingface
    volumes:
      - /data/models:/models:ro
      - /root/.cache/huggingface:/root/.cache/huggingface
    command:
      - --model
      - /models/DeepSeek-V4-Flash-0731
      - --served-model-name
      - deepseek-ai/DeepSeek-V4-Flash-0731
      - --trust-remote-code
      - --tensor-parallel-size
      - "4"
      - --enable-expert-parallel
      - --moe-backend
      - auto
      - --kv-cache-dtype
      - fp8
      - --block-size
      - "256"
      - --max-model-len
      - "1048576"
      - --max-num-seqs
      - "8"
      - --max-num-batched-tokens
      - "8192"
      - --gpu-memory-utilization
      - "0.95"
      - --tokenizer-mode
      - deepseek_v4
      - --reasoning-parser
      - deepseek_v4
      - --enable-auto-tool-choice
      - --tool-call-parser
      - deepseek_v4
      - --api-key
      - sk-123456
    ports:
      - "8000:8000"
    shm_size: "32g"
    ipc: host
    ulimits:
      memlock: -1
      stack: 67108864
    restart: unless-stopped
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

> 关键参数说明：
>
> * `--tensor-parallel-size 4`：4 卡并行。**换机器卡数不同必须同步改**。
>
> * `--enable-expert-parallel`：专家并行，MoE 推荐。
>
> * `--kv-cache-dtype fp8`：FP8 KV cache，省显存。
>
> * `--max-model-len 1048576`：1M 上下文。显存不足时降（如 262144）。
>
> * `--tokenizer-mode/--reasoning-parser/--tool-call-parser deepseek_v4`：DeepSeek V4 专用。
>
> * `--api-key sk-123456`：服务鉴权；建议交付前换成自定义随机 key。
>
> * `shm_size 32g` + `ipc: host` + `ulimits memlock -1`：大模型推理必须。

### 6.2 启动服务

```bash
cd /data/deploy
docker compose up -d
```

### 6.3 等待模型就绪并验证

大模型首启需加载权重 + JIT warmup，等待法：

```bash
# 端点就绪即成功（带鉴权）
until curl -s http://127.0.0.1:8000/v1/models -H "Authorization: Bearer sk-123456"; do
  echo "waiting..."; sleep 5
done
```

查看启动日志与显存：

```bash
docker logs -f deepseek-v4-flash-0731
nvidia-smi
```

### 6.4 对话测试

```bash
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" -H "Authorization: Bearer sk-123456" \
  -d '{"model":"deepseek-ai/DeepSeek-V4-Flash-0731","messages":[{"role":"user","content":"1+1=? 只回答数字"}],"max_tokens":50}'
```

**期望**：返回 `choices[0].message.content` = "2"，`system_fingerprint` 含 `vllm-0.27.1-tp4-ep`。

***

## 七、日常运维（启动/停止/重建）

```bash
cd /data/deploy
docker compose up -d          # 启动/重建（后台）
docker compose start          # 启动已存在容器
docker compose stop           # 停止（保留容器与权重）
docker compose restart        # 重启
docker compose down           # 停止并删除容器（权重卷不受影响）
# 改配置后必须重建：
docker compose down && docker compose up -d
# 直接操作单个容器：
docker stop deepseek-v4-flash-0731
docker start deepseek-v4-flash-0731
docker restart deepseek-v4-flash-0731
```

> `restart: unless-stopped`：宿主机重启后自动拉起。改参数须 `down+up`，`restart` 不读 yml。

***

## 八、交付验收清单（逐步执行）

在目标机执行以下命令，全部通过即可交付：

```bash
# 版本
docker --version                  # Docker 29.7.2
docker compose version            # v5.5.0
python3 --version                 # 3.12.3
node --version && npm --version   # v20.20.2 / 10.8.2
redis-cli ping                    # PONG
redis-cli info server | grep redis_version   # 7.0.15
/usr/local/bin/rdb --version      # usage: rdb [options] /path/to/dump.rdb

# 镜像
docker images | grep -i vllm      # 两个 vLLM 镜像都在

# GPU
docker run --rm --gpus all busybox nvidia-smi | head -1   # NVIDIA H200

# 服务
docker ps | grep deepseek                     # deepseek-v4-flash-0731 Up
curl -s http://127.0.0.1:8000/v1/models -H "Authorization: Bearer sk-123456"   # 200
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8000/v1/models       # 401（无鉴权）
nvidia-smi --query-gpu=index,memory.used,memory.total,utilization.gpu --format=csv
```

***

## 附：remote.py 免交互 SSH 工具

本机 `deploy-192.168.19.115/remote.py` 用于免交互在远端执行/传文件（无需手动输密码）。用法：

```bash
# 远程执行命令
python remote.py cmd "docker ps"
# 上传本地脚本到远端 /tmp
python remote.py upload ./install_docker.sh /tmp/install_docker.sh --sudo
```

> 密码写在脚本内（本机使用）。若换目标机，在 `remote.py` 内改 host/user/password 即可复用。

***

## 附：常见问题速查

| 现象                                   | 原因                                      | 处理                                                 |
| ------------------------------------ | --------------------------------------- | -------------------------------------------------- |
| 拉 vLLM 镜像极慢/失败                       | 国内无镜像加速                                 | 配置 daemon.json 镜像加速 + 代理重试                         |
| 容器无 GPU                              | nvidia-container-toolkit 未装/runtime 未配置 | 跑 install\_toolkit.sh 并重启 docker                   |
| 模型续传报 ImportError snapshot\_download | 用了系统 python3                            | 改用 `/opt/modelscp/bin/python3`                     |
| 启动 OOM 或显存不足                         | TP 与显存不匹配                               | 降 `--max-model-len` 或调整 `--gpu-memory-utilization` |
| 下载卡住 0MB                             | TCP 连接半开                                | kill 进程后重跑 download\_model\_robust.sh（自动续传）        |
| `/v1/models` 返回 401                  | 缺鉴权 key                                 | 请求头加 `Authorization: Bearer <api-key>`             |

