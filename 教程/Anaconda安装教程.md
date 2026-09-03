# Anaconda 安装教程（Ubuntu 26.04 LTS）

## 一、系统环境

| 项目 | 信息 |
|------|------|
| 操作系统 | Ubuntu 26.04 LTS |
| 用户 | liugroup |
| Anaconda 版本 | 2025.12（conda 25.11.1） |
| Python 版本 | 3.13.9 |

---

## 二、安装步骤

### 1. 更新系统 & 安装依赖

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install wget bzip2 libgl1 libx11-6 -y
```

### 2. 下载安装脚本

> 前往 https://repo.anaconda.com/archive/ 确认最新版本号。

```bash
cd /tmp
wget https://repo.anaconda.com/archive/Anaconda3-2025.12-2-Linux-x86_64.sh
```

### 3. 校验完整性（推荐）

```bash
sha256sum Anaconda3-2025.12-2-Linux-x86_64.sh
```

将输出与官网 Archive 页面提供的 SHA256 值比对。

### 4. 运行安装脚本

```bash
bash Anaconda3-2025.12-2-Linux-x86_64.sh
```

安装过程中注意：

| 提示 | 操作 |
|------|------|
| 许可协议 | 按 `Enter` 阅读，输入 `yes` 同意 |
| 安装路径 | 默认 `~/anaconda3`，直接回车 |
| **`conda init`?** | **必须输入 `yes`** |

### 5. 激活环境变量

```bash
source ~/.bashrc
```

### 6. 验证安装

```bash
conda --version
python --version
```

看到 `(base)` 前缀且版本号正常输出即安装成功。

---

## 三、常见问题：`conda: command not found`

### 现象

安装完成后执行 `conda --version` 提示：

```
conda: command not found
```

### 原因

安装时未执行 `conda init`，导致 `~/.bashrc` 中没有写入 conda 初始化代码。

### 排查

```bash
# 确认 conda 二进制文件存在
ls ~/anaconda3/bin/conda

# 检查 .bashrc 中是否有 conda 配置
grep -c "conda" ~/.bashrc
# 返回 0 说明未初始化
```

### 修复方法

#### 方法一：手动执行 conda init（推荐）

```bash
~/anaconda3/bin/conda init bash
source ~/.bashrc
```

#### 方法二：手动追加初始化代码

如果方法一不生效，手动写入 `~/.bashrc`：

```bash
cat >> ~/.bashrc << 'EOF'

# >>> conda initialize >>>
__conda_setup="$HOME/anaconda3/bin/conda"
if [ -f "$__conda_setup" ]; then
    eval "$("$__conda_setup" shell.bash hook)"
fi
unset __conda_setup
# <<< conda initialize <<<
EOF

source ~/.bashrc
```

### 验证修复

```bash
conda --version   # 应输出 conda 25.11.1
python --version  # 应输出 Python 3.13.9
```

终端提示符前出现 `(base)` 即表示成功。

---

## 四、安装后配置（可选）

### 配置国内镜像源（加速下载）

```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
conda config --set show_channel_urls yes
```

### 关闭自动激活 base 环境

```bash
conda config --set auto_activate_base false
```

### 创建独立项目环境

```bash
conda create -n myproject python=3.11
conda activate myproject
```

### GPU 支持（CUDA 13.2）

```bash
conda install nvidia/label/cuda-13.2.1::cuda-toolkit
```

---

## 五、卸载 Anaconda

```bash
rm -rf ~/anaconda3
# 手动删除 ~/.bashrc 中 conda initialize 相关代码块
nano ~/.bashrc
```

---

## 六、注意事项（Ubuntu 26.04）

- **Wayland 兼容**：若 `anaconda-navigator` 图形界面异常，切换至 Xorg 或设置 `export GDK_BACKEND=x11`。
- **python 命令**：Ubuntu 26.04 默认无 `python` 命令，conda 激活后自带，无需额外安装 `python-is-python3`。
- **ARM64 架构**：如为 aarch64 平台，请下载对应的 `aarch64` 安装脚本。
