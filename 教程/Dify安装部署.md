# Dify 安装部署记录（192.168.18.133）

## 概述

- 目标主机：`192.168.18.133`（Ubuntu 22.04.5 LTS x86\_64，128 核 / 125G 内存 / 1.8T 磁盘）
- 系统用户：`ps`（属 sudo 组）
- 部署产物：Dify v1.17.0（官方 docker-compose 方式）
- 访问地址：`http://192.168.18.133`（80 端口），HTTPS 为 443（本机未配置证书，建议走 HTTP）
- 首次访问需注册管理员账号后进入设置中心。

## 一、前置：安装 Docker 与 Docker Compose

服务器原本未装 Docker，通过阿里云源安装（官方源 download.docker.com 在国内连接不稳定）：

```
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://mirrors.aliyun.com/docker-ce/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://mirrors.aliyun.com/docker-ce/linux/ubuntu $(. /etc/os-release && echo $VERSION_CODENAME) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

验证：`docker --version`、`docker compose version`。
安装结果：Docker 29.7.2、Docker Compose v5.5.0。

## 二、配置国内镜像加速 + docker 组

写 `/etc/docker/daemon.json` 后 `sudo systemctl restart docker`：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://docker.1ms.run",
    "https://dockerproxy.net",
    "https://hub-mirror.c.163.com"
  ],
  "log-driver": "json-file",
  "log-opts": {"max-size": "20m", "max-file": "3"}
}
```

将 `ps` 加入 docker 组（`sudo usermod -aG docker ps`），重新登录后免 sudo。已用 `hello-world` 验证拉取与运行正常。

## 三、拉取 Dify 源码（docker 目录）

GitHub 大仓库 HTTP/2 连接易中断，采用浅克隆 + 稀疏检出，只取 docker 目录：

```
git config --system http.version HTTP/1.1
git clone --depth 1 --filter=blob:none --sparse https://github.com/langgenius/dify.git /opt/dify/dify
cd /opt/dify/dify
git sparse-checkout set docker
```

Commit：`fe39211`（2026-09-01，v1.17.0）。

## 四、配置 .env 并启动

```
cd /opt/dify/dify/docker
cp .env.example .env
# 若 SECRET_KEY 为空则生成：sed -i "s/^SECRET_KEY=$/SECRET_KEY=$(openssl rand -hex 32)/" .env
docker compose up -d
```

默认监听：nginx 80/443（80 端口原为空闲，直接占用）；plugin\_daemon 5003。

> 注意：默认 profile 下 weaviate/nginx 等由 `docker compose up -d` 一次拉起。若因并发 profile 出现 `Container /docker-weaviate-1 already in use` 冲突，可先 `docker compose down` 再 `up -d`。

## 五、运行状态确认

`docker compose ps` 显示全部容器 Up，其中 api / postgres / redis / sandbox 均 healthy：

| 服务                                                                       | 镜像                               | 健康状态           |
| ------------------------------------------------------------------------ | -------------------------------- | -------------- |
| nginx                                                                    | nginx:latest                     | Up，80/443 端口映射 |
| web                                                                      | langgenius/dify-web:1.17.0       | Up             |
| api                                                                      | langgenius/dify-api:1.17.0       | Up (healthy)   |
| worker                                                                   | langgenius/dify-api:1.17.0       | Up             |
| db\_postgres                                                             | postgres:15-alpine               | Up (healthy)   |
| redis                                                                    | redis:6-alpine                   | Up (healthy)   |
| weaviate                                                                 | semitechnologies/weaviate:1.27.0 | Up             |
| plugin\_daemon                                                           | langgenius/dify-plugin-daemon    | Up (5003)      |
| sandbox / local\_sandbox / agent\_backend / ssrf\_proxy / api\_websocket | 相关镜像                             | Up             |

验证：`curl -I http://localhost` 返回 `HTTP/1.1 307`（nginx 正常，跳转前端）；`curl http://localhost/health` 返回 Dify Web（Next.js）完整页面。

## 六、日常管理命令（需在 /opt/dify/dify/docker 下执行）

```bash
docker compose ps            # 查看状态
docker compose logs -f api   # 跟随后端日志
docker compose restart       # 重启全部
docker compose up -d         # 更新/拉起容器（先改 .env 或 pull 新镜像）
docker compose down          # 停止并移除容器/网络（保留卷）
docker compose down -v       # 连同数据卷一并删除（谨慎）
```

升级 Dify：`cd /opt/dify/dify && git sparse-checkout set docker && cd docker && docker compose pull && docker compose up -d`

## 相关脚本（本目录）

- `sshrun.py` / `sshscript.py` —— 本机 paramiko 远程执行工具（连接 192.168.18.133 / ps 用户）
- `install_docker.sh` —— 阿里云源安装 Docker + Compose
- `docker_mirror.sh` —— 配置镜像加速 / docker 组 / 拉取验证
- `clone_dify.sh` —— 浅克隆 + 稀疏检出 Dify docker 目录
- `deploy_dify.sh` / `redeploy.sh` —— 生成 .env、SECRET\_KEY、compose 拉起（冲突时 down 后 up）
- `inspect_config.sh` / `check_progress.sh` / `check_final.sh` —— 部署过程检查

