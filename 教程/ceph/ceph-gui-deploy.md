# Ceph Web 界面部署文档（File Browser + Filestash）

> **适用集群**：Ceph Squid 19.x + Ubuntu 24.04，4 节点（ceph1/2/3/5）
> **文档定位**：在 ceph1 上部署两个 Web 界面——
> ① **File Browser**（:8080）：挂载 CephFS 共享目录的文件管理器
> ② **Filestash**（:8334）：连接 RGW 的 S3 文件管理器
> **前置文档**：`ceph部署-副本模式.md`（集群部署）、`cephfs-guide.md`（CephFS 挂载）、`rgw-guide.md`（RGW/S3 部署）
> **更新日期**：2026-08-04

---

## 目录

- [一、部署总览](#一部署总览)
- [二、前置条件](#二前置条件)
- [三、File Browser（CephFS 文件管理）](#三file-browsercephfs-文件管理)
- [四、Filestash（S3 文件管理）](#四filestashs3-文件管理)
- [五、常见问题](#五常见问题)

---

## 一、部署总览

```
┌─ File Browser (:8080) ── 根目录 /mnt/cephfs ──→ CephFS（ceph-fuse 挂载）
│      文件上传/下载/预览，多节点共享同一目录
│
└─ Filestash (:8334) ───── Endpoint :80 ──────→ RGW（S3 API）
        S3 存储桶/对象管理，浏览器拖拽操作
```

两个 Web 界面**相互独立**：File Browser 管文件系统（CephFS），Filestash 管对象存储（S3/RGW），可只部署其中一个。

---

## 二、前置条件

- [ ] 集群正常：`ceph -s` 显示 `HEALTH_OK`
- [ ] **CephFS 已挂载**到 `/mnt/cephfs`（`mount | grep cephfs` 有输出；若无，先按 `cephfs-guide.md` 部署 ceph-fuse + `cephfs-mount.service`）
- [ ] **RGW 已部署**（`ceph orch ps --daemon-type rgw` 有 running 实例），且已有 S3 用户及 AK/SK（Filestash 需要）
- [ ] 可 SSH 登录 ceph1（root）

---

## 三、File Browser（CephFS 文件管理）

轻量级 Web 文件管理器，根目录指向 CephFS 共享目录，多节点写入的文件立即可见。

> **执行位置**：以下命令均在 **ceph1** 上执行。

### 3.1 下载并安装

```bash
# 方法一：ceph1 直连 GitHub（网络可达时）
curl -fsSL https://github.com/filebrowser/filebrowser/releases/latest/download/linux-amd64-filebrowser.tar.gz -o /tmp/fb.tar.gz

# 方法二：GitHub 被墙时，经 gh-proxy 代理下载
curl -fsSL "https://gh-proxy.com/https://github.com/filebrowser/filebrowser/releases/download/v2.63.23/linux-amd64-filebrowser.tar.gz" -o /tmp/fb.tar.gz

# 方法三：本机下载后 SFTP 上传到 ceph1:/tmp/fb.tar.gz

# 解压安装
tar -xzf /tmp/fb.tar.gz -C /usr/local/bin/
chmod +x /usr/local/bin/filebrowser
/usr/local/bin/filebrowser version
# 应输出：File Browser v2.63.23/xxxx
```

### 3.2 初始化配置

```bash
# 初始化数据库（根目录 /mnt/cephfs，端口 8080）
filebrowser config init --root=/mnt/cephfs --address=0.0.0.0 --port=8080 \
  --log=/var/log/filebrowser.log --database=/etc/filebrowser.db

# 放宽密码长度（默认要求 ≥12 位；如要短密码可设小）
filebrowser config set --minimumPasswordLength=6 --database=/etc/filebrowser.db

# 创建登录用户（用户名 admin，密码自定）
filebrowser users add admin Ps.123456 --perm.admin=true --database=/etc/filebrowser.db
```

### 3.3 注册 systemd 服务（开机自启）

```bash
cat > /etc/systemd/system/filebrowser.service << 'EOF'
[Unit]
Description=File Browser for CephFS
After=network.target

[Service]
ExecStart=/usr/local/bin/filebrowser -d /etc/filebrowser.db -r /mnt/cephfs
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable filebrowser.service
systemctl start filebrowser.service
```

### 3.4 验证

```bash
systemctl is-active filebrowser.service   # 应输出 active
ss -tlnp | grep 8080                      # 8080 端口监听
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8080/   # 200
```

**✔ 浏览器访问**：`http://192.168.12.176:8080`，账号 `admin` / 密码 `Ps.123456`，应能看到 `/mnt/cephfs` 下的文件列表。

> ⚠ **前提**：`/mnt/cephfs` 必须已挂载（ceph-fuse）。若页面显示 "It feels lonely here..." 且目录为空，先 `mount | grep cephfs` 排查挂载（见 `cephfs-guide.md` 7.2 节）。

---

## 四、Filestash（S3 文件管理）

开源 S3 文件管理器，通过 Podman 容器部署，支持浏览器拖拽管理 RGW 存储桶和对象。

> **执行位置**：以下命令均在 **ceph1** 上执行。

### 4.1 拉取镜像

```bash
# 拉取 Filestash 镜像（网络受限时可先 export http_proxy=https_proxy=代理地址）
podman pull docker.io/machines/filestash:latest
```

### 4.2 创建持久化卷并启动容器

```bash
# 配置持久化卷（保存管理员密码和 S3 连接配置，容器重建不丢失）
podman volume create filestash-data

# 启动容器（端口 8334）
podman run -d --name filestash --restart=always \
  -p 8334:8334 \
  -v filestash-data:/app/data \
  docker.io/machines/filestash:latest

podman ps --filter name=filestash --format "{{.Names}} {{.Status}} {{.Ports}}"
# 应输出：filestash Up X hours 0.0.0.0:8334->8334/tcp
```

### 4.3 注册 systemd 服务（推荐，更可靠的开机自启）

```bash
# 用 podman 自动生成标准 systemd 单元
podman generate systemd --name filestash --files --new
# 生成 container-filestash.service

cp container-filestash.service /etc/systemd/system/
systemctl daemon-reload
systemctl enable container-filestash.service

# 切换由 systemd 管理（先停容器再通过服务启动）
podman stop filestash && podman rm filestash
systemctl start container-filestash.service
```

### 4.4 首次配置（浏览器操作）

**✔ 浏览器访问**：`http://192.168.12.176:8334`

1. **设置管理员密码**：自动跳转 `/admin/setup`，输入管理员密码（保存到持久化卷）
2. **添加 S3 存储**：管理面板 → Storage → + Add new storage，类型选 **S3**：

| 字段 | 值 |
|------|-----|
| Endpoint | `http://192.168.12.176:80` |
| Bucket | `test-bucket` |
| Access Key | 你的 S3 用户 AK |
| Secret Key | 你的 S3 用户 SK |
| Region | 留空 |
| Path style | ✅ 勾选 |

3. 保存后即可在浏览器中浏览 / 上传 / 下载 / 搜索桶内对象。

### 4.5 验证

```bash
systemctl is-active container-filestash.service   # active
ss -tlnp | grep 8334                              # 8334 端口监听
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8334/   # 200（跳转 setup）
```

---

## 五、常见问题

| # | 问题 | 解决 |
|---|------|------|
| 1 | File Browser 下载 PDF 返回 ZIP 压缩包 | 🔧 v2.63.x 的下载接口用**路径参数**格式 `/api/raw/<路径>?inline=true`，前端 SPA 默认使用正确格式；手动测试勿用 `?path=` 查询参数 |
| 2 | File Browser 登录返回 401 | JWT 过期：浏览器 F12 → Application → Local Storage 清空该站点，重新登录 |
| 3 | File Browser 页面空白 / 无文件 | `mount \| grep cephfs` 确认挂载；`lsmod \| grep fuse` 确认 fuse 模块；`systemctl restart cephfs-mount.service` |
| 4 | Filestash 访问跳转死循环 / 打不开 | 不要设置 `APPLICATION_URL` 环境变量（会拼出错误地址），去掉后重启容器 |
| 5 | Filestash 容器重建后配置丢失 | 必须挂载 `-v filestash-data:/app/data` 持久化卷 |
| 6 | GitHub 下载 filebrowser 失败 | 用 gh-proxy 代理下载或本机下载后 SFTP 上传（见 3.1） |
| 7 | Filestash 连接 S3 报错 | 确认 Endpoint 带端口 80、勾选 Path style、AK/SK 正确 |

---

## 参考信息

| 项目 | 位置 |
|------|------|
| 集群部署文档 | `ceph部署-副本模式.md` |
| CephFS 挂载与使用 | `cephfs-guide.md` |
| RGW 对象存储 | `rgw-guide.md` |
| GUI 入口汇总 | `ceph-gui-summary.md` |
| cpolar 内网穿透 | `ceph-cpolar.md` |
