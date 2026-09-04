# Docker 安装部署

> 适用于：Ubuntu 系统。通过 Docker 官方软件源安装 Docker 引擎与 Docker Compose 插件，配置国内镜像加速与 DNS，最后用 nginx 容器验证部署是否成功。

以下命令需以 `sudo`（或 root 用户）执行：

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 创建密钥存放目录
sudo install -m 0755 -d /etc/apt/keyrings
# 下载docker gpg密钥
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 添加 Docker 软件源
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update

# 安装 Docker 引擎 + Compose V2 插件
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": [
    "https://docker.xuanyuan.me",
    "https://docker.1ms.run",
    "https://docker.m.daocloud.io"
  ],
  "dns": ["223.5.5.5","114.114.114.114"]
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证版本
docker -v
docker compose version

# 设置开机自启
sudo systemctl enable --now docker
# 查看状态
sudo systemctl status docker

# 拉取nginx
sudo docker pull nginx
#启动
sudo docker run --rm -p 8081:80 nginx
# 访问8080端口
192.168.12.164:8081
```

