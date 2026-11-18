# Docker 部署记录 — 192.168.18.116

> 部署日期：2026-09-30
> 目标机：Ubuntu 24.04.5 LTS，账号 `ide`（具备 sudo 权限）
> 目标：安装 Docker CE，配置国内镜像加速；**程序装系统盘、镜像与数据存大容量固态**

---

## 一、需求与磁盘规划

| 用途 | 路径 | 所在磁盘 | 容量 |
|---|---|---|---|
| Docker 程序（二进制） | `/usr/bin/docker`、`/usr/bin/dockerd` | nvme0n1p2（`/`） | 500G |
| Docker 数据根目录 `data-root` | `/mnt/nvme0n1p1/docker` | nvme1n1p1 | 3.7T |
| containerd 镜像层 / 快照 | `/mnt/nvme0n1p1/containerd` | nvme1n1p1 | 3.7T |

磁盘原始布局（部署前）：

```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0  7.3T  0 disk
└─sda1        8:1    0  7.3T  0 part /mnt/sda
nvme0n1     259:0    0  3.7T  0 disk
├─nvme0n1p1 259:2    0    1G  0 part /boot/efi
├─nvme0n1p2 259:3    0  500G  0 part /
└─nvme0n1p3 259:4    0  3.2T  0 part /home
nvme1n1     259:1    0  3.7T  0 disk
└─nvme1n1p1 259:5    0  3.7T  0 part /mnt/nvme0n1p1
```

> 注意：`/mnt/nvme0n1p1` 这个挂载点名字有误导性，它对应的实际是 **nvme1n1p1**（系统安装时被克隆所致）。已在 `/etc/fstab` 中按 UUID 永久挂载，重启不会丢。

部署前的环境检查：

- `/mnt/nvme0n1p1` 文件系统为 **xfs 且 `ftype=1`** → 满足 overlay2/overlayfs 存储驱动要求，无需额外处理
- 服务器可直连外网（阿里云、清华 TUNA、download.docker.com 均可达）

---

## 二、安装步骤

### 1. 配置 Docker CE 阿里云 apt 源

```bash
# 安装基础依赖
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

# 导入 GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 写入源
echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.asc] \
https://mirrors.aliyun.com/docker-ce/linux/ubuntu noble stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list
```

### 2. **先**写 `/etc/docker/daemon.json`，再装包

顺序很关键：如果先装包，Docker 服务会被自动拉起并把数据写入默认的 `/var/lib/docker`（系统盘），后续再改配置就得迁移数据。

```bash
sudo mkdir -p /mnt/nvme0n1p1/docker
sudo chmod 711 /mnt/nvme0n1p1/docker

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<'EOF'
{
  "data-root": "/mnt/nvme0n1p1/docker",
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me",
    "https://docker.1panel.live"
  ]
}
EOF
```

### 3. 安装组件

```bash
sudo apt-get update
sudo apt-get install -y \
  docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

安装版本：docker-ce `29.8.1`、containerd.io `2.3.6`、buildx `0.37.1`、compose `5.5.1`。

### 4. 清理默认目录并启动

```bash
sudo systemctl stop docker docker.socket
sudo rm -rf /var/lib/docker          # 若安装过程中已被自动创建
sudo systemctl enable --now containerd docker
```

---

## 三、关键坑：Docker 29 的 containerd 镜像存储

**这是本次部署最容易漏掉的一点。**

Docker 29 默认启用 **containerd 镜像存储**，`docker info` 显示：

```
 Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1
 Docker Root Dir: /mnt/nvme0n1p1/docker
```

此时镜像层**并不存在 `data-root` 里**，而是由系统 containerd 存在 `/var/lib/containerd`（**系统盘**，仅剩 468G）。实测拉取 hello-world 后：

| 目录 | 占用 | 说明 |
|---|---|---|
| `/mnt/nvme0n1p1/docker` | 152K | 仅元数据、volumes、buildkit |
| `/var/lib/containerd` | 232K | 真正的镜像内容与快照 |

因此需要把 containerd 的根目录也迁到大盘：

```bash
# 1. 停服务
sudo systemctl stop docker docker.socket containerd

# 2. 搬迁数据
sudo mkdir -p /mnt/nvme0n1p1/containerd
sudo cp -a /var/lib/containerd/. /mnt/nvme0n1p1/containerd/
sudo rm -rf /var/lib/containerd

# 3. 改配置（/etc/containerd/config.toml 中 root 行默认被注释）
sudo cp -a /etc/containerd/config.toml /etc/containerd/config.toml.bak.$(date +%Y%m%d%H%M%S)
sudo sed -i -E 's|^\s*#?\s*root\s*=.*|root = "/mnt/nvme0n1p1/containerd"|' /etc/containerd/config.toml
grep -n '^root' /etc/containerd/config.toml
# 输出：17:root = "/mnt/nvme0n1p1/containerd"

# 4. 启动
sudo systemctl daemon-reload
sudo systemctl start containerd
sudo systemctl start docker
```

迁移后已拉取的镜像无需重新下载，`docker images` 中仍可见。

---

## 四、验证结果

| 检查项 | 结果 |
|---|---|
| 服务状态 | `docker` / `containerd` / `docker.socket` 均 `enabled` + `active`（开机自启） |
| 版本 | Docker 29.8.1、Compose v5.5.1 |
| 数据落盘 | `Docker Root Dir: /mnt/nvme0n1p1/docker`，驱动 `overlayfs` |
| 镜像加速 | `docker info` 中 4 个 Registry Mirrors 已生效 |
| 拉取测试 | `alpine` 耗时 9 秒拉取成功 |
| 运行测试 | `docker run --rm hello-world` 输出 "Hello from Docker!" |
| 系统盘残留 | `/var/lib/docker`、`/var/lib/containerd` 均不存在 |

镜像源实测连通性（`/v2/` 返回 401/200 即表示可达）：

| 镜像源 | 结果 |
|---|---|
| `registry-1.docker.io`（官方直连） | **超时不可达** |
| `docker.m.daocloud.io` | 可达 |
| `docker.1ms.run` | 可达 |
| `docker.xuanyuan.me` | 可达 |
| `docker.1panel.live` | 可达 |
| `hub-mirror.c.163.com` | 不可达（已停服） |
| `dockerpull.org` | 不可达 |

---

## 五、常用命令

### 免 sudo 使用 Docker

```bash
sudo usermod -aG docker ide
# 需重新登录（退出 SSH 重连）或执行 newgrp docker 后生效
```

### 拉取与运行

```bash
# hello-world：最小验证
sudo docker pull hello-world
sudo docker run --rm hello-world

# nginx：验证拉取 + 运行 + 端口映射全链路
sudo docker pull nginx:latest
sudo docker run -d --name web -p 8080:80 nginx
# 浏览器访问 http://192.168.18.116:8080 应看到 Nginx 欢迎页
sudo docker rm -f web && sudo docker rmi nginx

# 轻量镜像，适合快速验证
sudo docker pull redis:7-alpine

# 基础系统镜像，便于进去试环境
sudo docker pull ubuntu:24.04
```

### 日常运维

```bash
docker info                       # 查看根目录、驱动、加速配置
docker images                     # 列出镜像
docker ps -a                      # 列出所有容器
docker system df                  # 查看磁盘占用
docker system prune -a            # 清理未使用的镜像/容器/网络（慎用）
du -sh /mnt/nvme0n1p1/containerd  # 镜像实际占用
du -sh /mnt/nvme0n1p1/docker      # Docker 元数据/卷占用
df -h /mnt/nvme0n1p1              # 大盘空间
```

### 修改镜像加速后生效

```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```

---

## 六、文件与备份清单

| 文件 | 说明 |
|---|---|
| `/etc/docker/daemon.json` | data-root + 镜像加速配置 |
| `/etc/containerd/config.toml` | containerd root 已指向大盘 |
| `/etc/containerd/config.toml.bak.*` | 修改前备份 |
| `/etc/apt/sources.list.d/docker.list` | Docker CE apt 源 |
| `/etc/apt/keyrings/docker.asc` | 阿里云镜像源 GPG key |

部署脚本留档（本地 `output/` 目录）：

- `setup_docker.sh` — 一体化安装 + 配置 + 验证
- `migrate_containerd.sh` — containerd 数据目录迁移
- `probe.sh` — 环境与镜像源探测
- `verify.sh` — 最终验证

---

## 七、注意事项

1. **`data-root` 与 containerd root 必须都在 nvme1n1p1**。只改 `daemon.json` 不够，Docker 29 的镜像数据主要在 containerd 侧，这是最容易踩的坑。
2. **重装/升级 Docker 后要复查一次** `docker info` 和 `/etc/containerd/config.toml`，确认配置未被覆盖。
3. `/mnt/nvme0n1p1` 挂载点命名与实际设备（nvme1n1）不符，排查时以 `lsblk` / `findmnt` 输出为准。
4. 系统盘 `/` 仅 500G，避免在其中保留镜像、构建缓存或数据卷。
5. 若哪天需要切回 Docker 官方源，`daemon.json` 的 `registry-mirrors` 改完后 `systemctl restart docker` 即可，无需重装。
