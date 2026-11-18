# 海光 BW100 ×2 部署 Qwen3.8-27B-FP8（vLLM）笔记

> 环境：麒麟 V10 + 海光 7390×2 + BW100×2（64GB×2）  
> 模型：`Qwen/Qwen3.8-27B-FP8`  
> 结论：使用海光 **vLLM 0.18** 定制镜像可成功启动 OpenAI 兼容 API

---

## 一、硬件与目录

| 项目 | 配置 |
|------|------|
| CPU | 海光 7390（32 核）×2，共 64 核 |
| 加速卡 | 海光 BW100（64GB）×2，共 128GB 显存 |
| 内存 | 256GB |
| 模型存放 | `/data/models/Qwen3.8-27B-FP8`（根分区 `/` 约 8T，勿放 `/tmp`） |

显卡检查：

```bash
hy-smi
```

---

## 二、模型下载（ModelScope）

系统自带 Python 3.7 过旧，用 Miniconda：

```bash
# 已有 llm 环境可跳过创建
conda activate llm
pip install modelscope -i https://pypi.tuna.tsinghua.edu.cn/simple

sudo mkdir -p /data/models
sudo chown -R $USER:$USER /data/models

# 不要加 sudo，否则找不到 conda 里的 modelscope
modelscope download --model Qwen/Qwen3.8-27B-FP8 \
  --local_dir /data/models/Qwen3.8-27B-FP8
```

---

## 三、关键：不要用这些镜像

| 镜像 / 方式 | 结果 |
|-------------|------|
| `vllm/vllm-openai:latest`（Docker Hub / NVIDIA） | BW100 不可用；且需配 Docker 代理才能拉 |
| `vllm0.8.5-...-dtk25.04` | 不认识 `qwen3_5` |
| `vllm0.15.1-...-dtk26.04` | 不支持 `Qwen3_5ForConditionalGeneration` |
| `sglang0.5.12-...-dtk26.04` | 启动时 KERNEL VMFault（exit -11） |

**可用镜像（已验证）：**

```text
harbor.sourcefind.cn:5443/dcu/admin/base/custom:vllm018-ubuntu22.04-dtk26.04-gemma4-0413
```

说明：该镜像面向海光 DTK 26.04 + Qwen3.5 同架构（`Qwen3_5`），与 Qwen3.8 兼容。

---

## 四、拉取镜像并启动容器

**宿主机执行：**

```bash
docker pull harbor.sourcefind.cn:5443/dcu/admin/base/custom:vllm018-ubuntu22.04-dtk26.04-gemma4-0413

docker rm -f vllm-qwen38 2>/dev/null

docker run -it --name vllm-qwen38 \
  --shm-size=64g \
  --network=host \
  --privileged \
  --device=/dev/kfd \
  --device=/dev/dri \
  --device=/dev/mkfd \
  --group-add video \
  --cap-add=SYS_PTRACE \
  --security-opt seccomp=unconfined \
  -v /opt/hyhal:/opt/hyhal:ro \
  -v /data/models/Qwen3.8-27B-FP8:/model \
  --entrypoint bash \
  harbor.sourcefind.cn:5443/dcu/admin/base/custom:vllm018-ubuntu22.04-dtk26.04-gemma4-0413
```

要点：

- 必须挂 `/opt/hyhal`、`/dev/kfd`、`/dev/dri`、`/dev/mkfd`
- 用 `--network=host`，端口直接映射到宿主机
- **不要**使用 NVIDIA 的 `--gpus all`

---

## 五、容器内启动 vLLM

进入容器后（提示符为 `root@...`）：

```bash
export HIP_VISIBLE_DEVICES=0,1

vllm serve /model \
  --served-model-name Qwen3.8-27B \
  --tensor-parallel-size 2 \
  --dtype auto \
  --max-model-len 256000 \
  --gpu-memory-utilization 0.92 \
  --trust-remote-code \
  --reasoning-parser qwen3 \
  --host 0.0.0.0 \
  --port 8000
```

成功标志（日志中可见）：

```text
Starting vLLM server on http://0.0.0.0:8000
Application startup complete.
```

首启加载模型可能要几分钟，**不要急着 Ctrl+C**。

---

## 六、验证接口

另开**宿主机**终端：

```bash
curl http://127.0.0.1:8000/v1/models
```

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "Qwen3.8-27B",
    "messages": [{"role": "user", "content": "你好，用一句话介绍你自己"}],
    "max_tokens": 128
  }'
```

常用路由：`/v1/models`、`/v1/chat/completions`、`/health`。

---

## 七、可选：后台运行与固化

容器内后台：

```bash
nohup vllm serve /model \
  --served-model-name Qwen3.8-27B \
  --tensor-parallel-size 2 \
  --dtype auto \
  --max-model-len 4096 \
  --gpu-memory-utilization 0.85 \
  --trust-remote-code \
  --reasoning-parser qwen3 \
  --host 0.0.0.0 \
  --port 8000 > /tmp/vllm.log 2>&1 &

tail -f /tmp/vllm.log
```

环境调通后可提交镜像，避免每次重配：

```bash
# 宿主机
docker commit vllm-qwen38 vllm-qwen38:local
```

下次可直接：

```bash
docker start -ai vllm-qwen38
# 或用 vllm-qwen38:local 重新 run
```

---

## 八、参数说明（当前配置）

| 参数 | 值 | 说明 |
|------|-----|------|
| `--tensor-parallel-size` | 2 | 两张 BW100 张量并行 |
| `--max-model-len` | 4096 | 保守上下文，稳妥优先；显存够可加大 |
| `--gpu-memory-utilization` | 0.85 | 预留部分显存 |
| `--reasoning-parser` | qwen3 | 解析思考链相关输出 |
| `--served-model-name` | Qwen3.8-27B | API 请求里的 `model` 字段 |

---

## 九、参考

- 海光推理 cookbook：https://github.com/HYGON-AI/dcu-inference-cookbook  
- 光合 Qwen3.5 ModelZoo（vLLM 0.18 镜像来源）：https://developer.sourcefind.cn/codes/modelzoo/qwen3.5  

---

*记录日期：2026-09-30*  
*状态：vLLM 服务已成功启动（Application startup complete）*
