# ComfyUI 四卡部署完整指南

> 适用环境：Ubuntu 24.04 + 4× NVIDIA A100（GPU 0/1/2/3）+ Python 3.12
> 目标：部署 4 个独立 ComfyUI 实例，每个实例隔离绑定一张 A100

## 目录

- [0. 前置：确认 4 张 A100 和驱动](#0-前置确认-4-张-a100-和驱动)
- [1. 如果还没装驱动：安装 NVIDIA 驱动](#1-如果还没装驱动安装-nvidia-驱动)
- [2. 安装系统依赖 + 创建目录](#2-安装系统依赖--创建目录)
- [3. 克隆 4 份 ComfyUI 并各建独立 Python 环境](#3-克隆-4-份-comfyui-并各建独立-python-环境)
- [4. 每个环境安装 PyTorch(CUDA版) 和 ComfyUI 依赖](#4-每个环境安装-pytorchecuda版-和-comfyui-依赖)
- [5. 先手动启动第 1 个实例，验证单卡能跑](#5-先手动启动第-1-个实例验证单卡能跑)
- [6. 创建 4 个启动脚本（各自绑定一张卡 + 不同端口）](#6-创建-4-个启动脚本各自绑定一张卡--不同端口)
- [7. systemd 服务化（开机自启 + 崩溃自动拉起）](#7-systemd-服务化开机自启--崩溃自动拉起)
- [8. 验证全部是否正常 + 绑定是否正确](#8-验证全部是否正常--绑定是否正确)
- [9. 开启远程访问（可选，浏览器在别处访问）](#9-开启远程访问可选浏览器在别处访问)
- [10. 模型文件放置位置与下载](#10-模型文件放置位置与下载)
- [11. 4 个实例共享模型（软链接）](#11-4-个实例共享模型软链接)
- [常用维护命令](#常用维护命令)
- [常见问题与故障排查](#常见问题与故障排查)

---

## 0. 前置：确认 4 张 A100 和驱动

```bash
sudo apt update
nvidia-smi
```

- 看到 `GPU 0/1/2/3` 共 4 张 A100 → 驱动已就绪，跳到第 2 步
- 提示 `command not found` 或没有显卡 → 先装驱动（见第 1 步）

## 1. 如果还没装驱动：安装 NVIDIA 驱动

```bash
# Ubuntu 24.04 用 550 系列（A100 支持良好）
sudo apt install -y nvidia-driver-550
# 执行 nvidia-smi 确认 4 张卡
nvidia-smi
```

## 2. 安装系统依赖 + 创建目录

```bash
sudo apt install -y git git-lfs ffmpeg libgl1 libglib2.0-0 python3.12-venv python3.12-dev build-essential

# 统一放代码的目录（当前用户赋予权限）
sudo mkdir -p /opt/comfy
sudo chown $USER:$USER /opt/comfy
```

> `python3.12-dev build-essential` 必须装：Triton 的 JIT 编译依赖 `Python.h` 和 gcc，缺失会导致报错（详见故障排查第 4 条）。

## 3. 克隆 4 份 ComfyUI 并各建独立 Python 环境

```bash
cd /opt/comfy

for i in 0 1 2 3; do
  # 官方仓库已迁移到 Comfy-Org，如需指定稳定版加 --branch v0.31.0
  git clone https://github.com/Comfy-Org/ComfyUI --branch v0.31.0 comfyui-$i
  python3.12 -m venv comfyui-$i/venv
done
```

> 显式使用 `python3.12 -m venv`，确保 4 个 venv 都基于 Python 3.12。

## 4. 每个环境安装 PyTorch(CUDA版) 和 ComfyUI 依赖

pip 会把下载好的 wheel 包缓存到本机，4 次循环**只会下载一次 torch，后面直接复用本地缓存**，不用重复拉几百 MB 文件:

```bash
cd /opt/comfy

# 设置pip缓存目录，所有venv共用缓存
export PIP_CACHE_DIR=/opt/comfy/pip_cache
mkdir -p $PIP_CACHE_DIR

for i in 0 1 2 3; do
  cd comfyui-$i
  ./venv/bin/pip install --upgrade pip -i https://pypi.tuna.tsinghua.edu.cn/simple
  ./venv/bin/pip install torch torchvision torchaudio \
    -i https://pypi.tuna.tsinghua.edu.cn/simple \
    --extra-index-url https://download.pytorch.org/whl/cu130
  ./venv/bin/pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
  cd /opt/comfy
done
```

> 如果 `cu130` 安装失败或兼容报错，把上面 `cu130` 换成 `cu126` 或 `cu124` 重试。

## 5. 先手动启动第 1 个实例，验证单卡能跑

```bash
cd /opt/comfy/comfyui-0
CUDA_VISIBLE_DEVICES=0 ./venv/bin/python main.py --listen 0.0.0.0 --port 8188
```

浏览器访问 `http://服务器IP:8188`，能打开界面就说明单实例 OK。`Ctrl+C` 停掉。

## 6. 创建 4 个启动脚本（各自绑定一张卡 + 不同端口）

```bash
for i in 0 1 2 3; do
cat > /opt/comfy/comfyui-$i/start.sh <<EOF
#!/bin/bash
export CUDA_VISIBLE_DEVICES=$i
cd /opt/comfy/comfyui-$i
exec ./venv/bin/python main.py --listen 0.0.0.0 --port $((8188+i)) \\
     --output-directory /opt/comfy/output-$i
EOF
  chmod +x /opt/comfy/comfyui-$i/start.sh
done
```

4 个实例对应：

| 实例      | 绑定显卡   | 端口 |
| --------- | ---------- | ---- |
| ComfyUI-0 | A100 GPU 0 | 8188 |
| ComfyUI-1 | A100 GPU 1 | 8189 |
| ComfyUI-2 | A100 GPU 2 | 8190 |
| ComfyUI-3 | A100 GPU 3 | 8191 |

## 7. systemd 服务化（开机自启 + 崩溃自动拉起）

```bash
for i in 0 1 2 3; do
sudo tee /etc/systemd/system/comfy-$i.service > /dev/null <<EOF
[Unit]
Description=ComfyUI instance $i on A100[$i]
After=network.target

[Service]
Type=simple
User=$USER
WorkingDirectory=/opt/comfy/comfyui-$i
Environment=CUDA_VISIBLE_DEVICES=$i
Environment="PYTHONUNBUFFERED=1"
ExecStart=/opt/comfy/comfyui-$i/venv/bin/python main.py --listen 0.0.0.0 --port $((8188+i)) --output-directory /opt/comfy/output-$i --enable-cors-header
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
done

sudo systemctl daemon-reload
sudo systemctl enable comfy-0 comfy-1 comfy-2 comfy-3
sudo systemctl start  comfy-0 comfy-1 comfy-2 comfy-3
```

> `--enable-cors-header` 是远程访问必需参数，缺少它从其他机器访问会报 403（详见第 9 节）。

## 8. 验证全部是否正常 + 绑定是否正确

```bash
# 四个服务都是 active (running)
systemctl status comfy-0 comfy-1 comfy-2 comfy-3

# 关键：确认 4 张卡底下各有一个独立的 python 进程
nvidia-smi

# 看 4 个进程各自的 PID 和启动端口
ps aux | grep main.py

# 浏览器分别访问（记得先开防火墙）
#   http://IP:8188 / 8189 / 8190 / 8191
```

开防火墙（若启用）：
```bash
sudo ufw allow 8188,8189,8190,8191/tcp
```

部署成功标志：`nvidia-smi` 中 4 张 A100 下各有一个独立的 python 进程，每张卡显存约 400 MB（待机状态）。GPU 利用率 0%、功耗 40W 左右均属正常待机。

## 9. 开启远程访问（可选，浏览器在别处访问）

ComfyUI 默认做 Host/Origin 校验，仅凭 `--listen 0.0.0.0` 从其他机器访问会被拦截返回 403。要开启远程访问需要两步：

**第 1 步：`ExecStart` 加 `--enable-cors-header`**

第 7 节的 systemd 配置已经内置了该参数。若你用的是旧配置，用 sed 批量补上：

```bash
for i in 0 1 2 3; do
  sudo sed -i "s/--output-directory \/opt\/comfy\/output-\$i/--output-directory \/opt\/comfy\/output-$i --enable-cors-header/" /etc/systemd/system/comfy-$i.service
done
sudo systemctl daemon-reload
sudo systemctl restart comfy-0 comfy-1 comfy-2 comfy-3
```

**第 2 步：放行防火墙/安全组端口**

```bash
sudo ufw allow 8188,8189,8190,8191/tcp
```

如果用的是云服务器，还要在**云控制台的安全组**里放行这 4 个端口。

**验证**：浏览器访问 `http://<服务器IP>:8188` 能打开界面即成功。

## 10. 模型文件放置位置与下载

### 核心规律

官方工作流推荐的目录结构（在 `comfyui-X/models/` 下）：

| 模型类型 | 放置目录 |
|----------|----------|
| 扩散主模型（UNet / Diffusion Model） | `models/diffusion_models/` |
| 文本编码器（CLIP / LLM，如 Qwen、Mistral） | `models/text_encoders/` |
| VAE（图像/视频解码器） | `models/vae/` |
| 加速 LoRA（xx_turbo、4step、8step） | `models/loras/` |
| 旧式大 checkpoint（可选） | `models/checkpoints/` |

### 已部署模型的放置对照

**Z-Image（文生图，实例 2，端口 8190）**

| 文件 | 目录 |
|------|------|
| `z_image_turbo_bf16.safetensors`（UNet 主模型） | `models/unet/` |
| `qwen_3.4b.safetensors`（CLIP 文本编码器） | `models/clip/` |
| `ae.safetensors`（VAE） | `models/vae/` |

> 注意：Z-Image 走的是原生节点旧式目录（`unet/`、`clip/`），与社区模板的 `diffusion_models/`、`text_encoders/` 不同。

**MiniMax-H3（视频，实例 2，端口 8190）**

| 文件 | 目录 |
|------|------|
| `minimax_h3_fl2va_pruned_int8_convrot.safetensors` | `diffusion_models/` |
| `minimax_h3_ref2va_pruned_int8_convrot.safetensors` | `diffusion_models/` |
| `minimax_h3_fl2v_turbo_8step_v1.0_comfyui_bf16.safetensors` | `loras/` |
| `minimax_h3_ref2v_turbo_4step_v0.1_comfyui_bf16.safetensors` | `loras/` |
| `qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors`（LLM 编码器） | `text_encoders/` |
| `minimax_h3_video_vae_fp16.safetensors` | `vae/` |
| `minimax_h3_audio_vae_fp32.safetensors` | `vae/` |

**Flux.2（图像，实例 2，端口 8190）**

| 文件 | 目录 |
|------|------|
| `flux2_dev_fp8mixed.safetensors`（主模型） | `diffusion_models/` |
| `mistral_3_small_flux2_bf16.safetensors`（文本编码器） | `text_encoders/` |
| `full_encoder_small_decoder.safetensors`（VAE） | `vae/` |
| `Flux_2-Turbo-LoRA_comfyui.safetensors`（加速 LoRA） | `loras/` |

### 国内镜像下载示例（免代理）

```bash
mkdir -p /opt/comfy/comfyui-0/models/vae
cd /opt/comfy/comfyui-0/models/vae
# 用 hf-mirror 镜像下载，国内直连快
wget -O ae.safetensors \
  "https://hf-mirror.com/Comfy-Org/z_image_turbo/resolve/main/split_files/vae/ae.safetensors"
```

## 11. 4 个实例共享模型（软链接）

想让 8188/8189/8190/8191 四个端口共用同一批模型，做软链接即可（避免每份模型重复占用几十 GB）：

```bash
# 以 comfyui-2 的 models 为基准，其他实例软链接过去
for i in 0 1 3; do
  rm -rf /opt/comfy/comfyui-$i/models
  ln -s /opt/comfy/comfyui-2/models /opt/comfy/comfyui-$i/models
done
sudo systemctl restart comfy-0 comfy-1 comfy-2 comfy-3
```

---

## 常用维护命令

```bash
# 查看某个实例日志
sudo journalctl -u comfy-0 -f

# 单独重启某个实例
sudo systemctl restart comfy-2

# 全部停止 / 启动
sudo systemctl stop  comfy-0 comfy-1 comfy-2 comfy-3
sudo systemctl start comfy-0 comfy-1 comfy-2 comfy-3

# 查看某个实例使用的 Python 版本
/opt/comfy/comfyui-0/venv/bin/python --version
```

## 常见问题与故障排查

- **模型放哪**：默认每个实例用自己的 `comfyui-X/models/` 目录。想共用一套模型，可在 `start.sh` 的 python 命令加 `--base-directory /opt/comfy/comfyui-0` 让其他实例读公共模型，但**输出目录保持独立**避免写冲突。
- **显存不够 OOM**：A100 显存充足一般不会，但内存建议 ≥32GB、CPU ≥16 核。
- **端口被占用**：把对应实例的 `--port` 改掉并 `daemon-reload` 重启。
- **Triton 编译报错（gcc / cuda_utils.c）**：日志出现 `Command ['/usr/bin/gcc', ..., '/usr/include/python3.12'] ... non-zero exit status 1`，是缺少 `python3.12-dev` 和 `build-essential`。执行：
  ```bash
  sudo apt install -y python3.12-dev build-essential
  rm -rf ~/.triton /tmp/triton_*   # 清理旧的失败编译缓存
  sudo systemctl restart comfy-0
  ```
- **报错 `Cannot handle this data type: (1, 1, 16)`（存图失败）**：`加载VAE` 节点选成了空 VAE（`pixel_space`），16 通道 latent 未解码成 RGB。需在 VAE 节点下拉框选真实的 VAE 文件（如 `ae.safetensors`）。
- **工作流报"缺失模型"**：模型文件放入目录后，**节点下拉框显示的文件名必须和目录中实际文件名一字不差**（含大小写、`.safetensors` 后缀）。改名或刷新页面（F5）重新扫描。
- **turbo LoRA 不生效**：`xx_turbo` 类权重属于 LoRA，放 `diffusion_models/` 会找不到，应放 `loras/`；且需在界面用 `加载LoRA` 节点手动选中连好。
- **端口访问打不开（403）**：未加 `--enable-cors-header` 或防火墙/云安全组未放行，见第 9 节。