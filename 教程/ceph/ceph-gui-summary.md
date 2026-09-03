# Ceph 集群图形化界面（GUI）汇总

> **集群环境**：Ubuntu 24.04 × 4 节点，Ceph Squid 19.x
> **服务器 IP**：192.168.12.176（ceph1）
> **更新日期**：2026-08-04

---

## 目录

- [一、Ceph Dashboard（集群管理）](#一ceph-dashboard集群管理)
- [二、File Browser（CephFS 文件管理）](#二file-browsercephfs-文件管理)
- [三、Filestash（S3 文件管理，推荐）](#三filestashs3-文件管理推荐)
- [四、公网访问（cpolar 内网穿透）](#四公网访问cpolar-内网穿透)
- [五、服务与端口汇总](#五服务与端口汇总)

---

## 一、Ceph Dashboard（集群管理）

Ceph 官方自带的 Web 管理面板，用于查看与管理集群状态，随 ceph-mgr 自动启动。

### 入口

| 项目 | 值 |
|------|-----|
| 访问地址 | `https://192.168.12.176:8443` |
| 用户名 | `admin` |
| 密码 | `Ps.123456` |

> ⚠ HTTPS 自签名证书，浏览器首次访问需「高级 → 继续前往」。

### 功能

仪表盘（健康/容量/IOPS）、主机、OSD、存储池、MON/MGR/MDS 监控、CephFS 状态、RGW 状态。

### 管理命令

```bash
# 查看状态 / 端口
systemctl status ceph-a2173549-84bf-11f1-8f8e-000c29c6cc12@mgr.ceph1
ss -tlnp | grep 8443

# 重置密码
echo "新密码" | cephadm shell -- ceph dashboard ac-user-set-password admin -i -
```

---

## 二、File Browser（CephFS 文件管理）

轻量 Web 文件管理器，根目录指向 CephFS 共享目录 `/mnt/cephfs`，支持浏览器上传/下载/预览。

### 入口

| 项目 | 值 |
|------|-----|
| 访问地址 | `http://192.168.12.176:8080` |
| 用户名 | `admin` |
| 密码 | `Ps.123456` |
| 根目录 | `/mnt/cephfs`（CephFS 共享存储） |

### 功能

文件上传/下载/删除/重命名、拖拽上传、目录管理、文本/PDF/图片在线预览、分享链接。

### 管理命令

```bash
systemctl start | stop | status | enable filebrowser.service
```

| 项目 | 值 |
|------|-----|
| 可执行文件 | `/usr/local/bin/filebrowser`（v2.63.23） |
| 数据库 | `/etc/filebrowser.db` |
| 日志 | `/var/log/filebrowser.log` |

---

## 三、Filestash（S3 文件管理，推荐）

开源 S3 文件管理器，Podman 容器部署，功能最完整的 S3 Web 界面。

### 入口

| 项目 | 值 |
|------|-----|
| 访问地址 | `http://192.168.12.176:8334` |
| 首次访问 | `/admin/setup` 设置管理员密码 |
| 后续访问 | 管理员密码登录 |

### S3 存储配置（管理后台添加）

| 字段 | 值 |
|------|-----|
| Endpoint | `http://192.168.12.176:80` |
| Bucket | `test-bucket` |
| Access Key | `3QQSL651T0L3UHILET23` |
| Secret Key | `VUVHSqbhrhXuBseKDSZTHZuRpdeVhCI0Bp4xRcwL` |
| Region | 留空 |
| Path style | ✅ 勾选 |

### 管理命令

```bash
systemctl start | stop | restart | status container-filestash.service
podman logs filestash
```

| 项目 | 值 |
|------|-----|
| 镜像 | `docker.io/machines/filestash:latest` |
| 端口映射 | `8334:8334` |
| 持久化卷 | `filestash-data → /app/data`（配置持久保存） |

---

## 四、公网访问（cpolar 内网穿透）

各服务已通过 cpolar 暴露公网（免费版 4 隧道，Dashboard 占 2 条）。

> ⚠ **公网地址为动态分配**，cpolar 每次重启/重连都会变化。获取当前地址：

```bash
grep "Tunnel established" /var/log/cpolar/access.log | tail -4
```

隧道配置（`/usr/local/etc/cpolar/cpolar.yml`）：

| 隧道名 | 协议 | 本地地址 | 对应服务 |
|--------|------|---------|---------|
| dashboard-tcp | tcp | 127.0.0.1:8443 | Ceph Dashboard（TCP 备用） |
| dashboard-web | http | 127.0.0.1:8444 | Ceph Dashboard（经 Nginx 反代） |
| filebrowser | http | 127.0.0.1:8080 | File Browser |
| filestash | http | 127.0.0.1:8334 | Filestash |

> Dashboard 因 HTTPS 自签名证书无法被 cpolar 直接代理，由 Nginx 监听 8444 反代到 8443（`proxy_ssl_verify off`）。详细配置见 `ceph-cpolar.md`。

---

## 五、服务与端口汇总

| 服务 | 端口 | 状态 | 管理方式 |
|------|------|------|---------|
| Ceph Dashboard | 8443 (HTTPS) | ✅ 自动 | ceph-mgr systemd |
| File Browser | 8080 (HTTP) | ✅ 运行 | `systemctl ... filebrowser` |
| Filestash | 8334 (HTTP) | ✅ 运行 | `systemctl ... container-filestash` |
| RGW（S3 API） | 80 (HTTP) | ✅ 运行 | ceph orch |
| Nginx 反代 | 8444 (HTTP) | ✅ 运行 | `systemctl ... nginx` |

---

## 参考信息

| 项目 | 位置 |
|------|------|
| cpolar 穿透配置 | `ceph-cpolar.md` |
| CephFS 专题 | `cephfs-guide.md` |
| RGW 对象存储专题 | `rgw-guide.md` |
| 集群部署文档 | `ceph部署-副本模式.md` |
