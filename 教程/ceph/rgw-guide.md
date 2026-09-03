# RGW 对象存储（S3 兼容 API）使用指南

> **适用集群**：Ceph Squid 19.x + Ubuntu 24.04，4 节点（ceph1/2/3/5），4 OSD，240 GiB
> **文档定位**：从部署 RGW 服务到客户端使用 S3 SDK 的操作手册（**基于实际部署验证**）
> **前置文档**：`ceph部署-副本模式.md` — 集群部署流程
> 标注 🔧 的章节为根据实际环境修正的内容。



## 目录

- [一、RGW 概述](#一rgw-概述)
- [二、前提条件](#二前提条件)
- [三、部署 RGW 服务](#三部署-rgw-服务)
- [四、创建 S3 用户与访问密钥](#四创建-s3-用户与访问密钥)
- [五、s3cmd 客户端](#五s3cmd-客户端)
- [六、AWS CLI](#六aws-cli)
- [七、Python SDK（boto3）](#七python-sdkboto3)
- [八、高级配置](#八高级配置)
- [九、故障排查](#九故障排查)
- [十、卸载与清理](#十卸载与清理)

---

## 一、RGW 概述

### 1.1 是什么

RGW（RADOS Gateway）是 Ceph 的**对象存储网关**，对外暴露兼容 **Amazon S3** 的 RESTful API。应用通过标准 HTTP/HTTPS 上传下载数据，无需安装 Ceph 专用客户端。

### 1.2 核心概念

| 概念 | S3 术语 | 说明 |
|------|---------|------|
| 用户 | User | 访问身份，每个用户有独立 Access Key / Secret Key |
| 存储桶 | Bucket | 对象的容器，类似文件夹但不可嵌套 |
| 对象 | Object | 存储基本单元 = 数据 + 元数据 + 唯一 Key |
| Access Key | AK | 用户标识（20 位字符） |
| Secret Key | SK | 访问密钥（40 位字符），用于签名请求 |

### 1.3 架构与存储位置

```
S3 客户端（s3cmd / AWS CLI / SDK）
        │ HTTP S3 API
        ▼
RGW 网关（rgw.myrgw，默认监听 :80）
        │ RADOS 原生协议
        ▼
RADOS 存储池（rgw_data 存对象内容 / rgw_meta、rgw_log 存元数据，3 副本分布所有节点）
```

> RGW 只是**协议翻译层**：把 HTTP 的 S3 API 翻译成底层 RADOS 调用。文件最终切片、副本、分布到多台节点的多块磁盘上（你通过 S3 上传的文件实际存在 `default.rgw.buckets.data` 等池里）。

### 1.4 三种存储对比

| 特性 | RBD（块） | CephFS（文件） | RGW（对象） |
|------|----------|---------------|-------------|
| 访问协议 | 内核块设备 | POSIX 文件系统 | **S3 (HTTP)** |
| 客户端要求 | 内核 rbd 模块 | ceph-fuse / 内核模块 | **任何 HTTP 客户端** |
| 多机共享 | ❌ | ✅ | ✅ |
| 典型延迟 | 微秒级 | 毫秒级 | 毫秒~百毫秒 |
| 适用场景 | 虚拟机、数据库 | 共享文件系统 | 静态文件、备份、API 存储 |

---

## 二、前提条件

- [x] Ceph 集群运行正常：`ceph -s` 返回 `HEALTH_OK`
- [x] 集群 OSD ≥ 3，容量充足
- [x] 各节点网络互通

### 本集群环境速查

| 项目 | 值 |
|------|-----|
| MON 地址 | `192.168.12.176` / `.90` / `.169` |
| 节点 / OSD 数 | 🔧 4（ceph1/2/3/5）/ 4，240 GiB |
| RGW 访问地址 | 🔧 `http://192.168.12.176`（Squid 默认端口 **80**） |
| RGW 服务名 | `myrgw` |
| RGW 数据池 | `rgw_data`（PG 128）/ `rgw_meta`、`rgw_log`（PG 32） |
| 默认 S3 用户 | `testuser` |

---

## 三、部署 RGW 服务

> **执行位置**：ceph1。

```bash
# 1. 创建存储池（可选，cephadm 也会自动建；数据池 128，元数据/日志池 32）
ceph osd pool create rgw_data 128 128
ceph osd pool create rgw_meta 32 32
ceph osd pool create rgw_log 32 32

# 2. 部署 RGW（三节点各一个实例）
ceph orch apply rgw myrgw --placement="3 ceph1 ceph2 ceph3"
```

**✔ 验证**：

```bash
ceph orch ps --daemon-type rgw
# 三行 rgw 实例，状态均 running

podman ps --format "{{.Names}} {{.Ports}}" | grep rgw
# 输出示例：ceph-a2173549...-rgw.myrgw.ceph1 0.0.0.0:80->8080/tcp
# 🔧 Squid 默认端口是 80（不是旧版 7480）
```

---

## 四、创建 S3 用户与访问密钥

> **执行位置**：ceph1。
> 🔧 本集群 `ceph-common` 未安装，`radosgw-admin` 需用 `cephadm shell -- radosgw-admin` 形式执行（下同）。

```bash
# 创建用户（记录输出中的 access_key 和 secret_key）
cephadm shell -- radosgw-admin user create --uid=testuser --display-name="Test User"

# 查看用户
cephadm shell -- radosgw-admin user list
cephadm shell -- radosgw-admin user info --uid=testuser

# 创建多个用户（可选）
cephadm shell -- radosgw-admin user create --uid=app1 --display-name="Application 1"

# 为已有用户创建子密钥（可选）
cephadm shell -- radosgw-admin key create --uid=testuser --key-type=s3
```

输出示例（JSON）：

```json
"keys": [ { "user": "testuser", "access_key": "ABCDEF...", "secret_key": "abcdef..." } ]
```

---

## 五、s3cmd 客户端

### 5.1 安装与配置

```bash
apt install -y s3cmd

# 交互配置，或直接写 ~/.s3cfg：
cat > ~/.s3cfg << 'CFG'
[default]
access_key = <你的 access_key>
secret_key = <你的 secret_key>
host_base = 192.168.12.176
host_bucket = 192.168.12.176
use_https = False
signature_v2 = False
CFG
```

> 🔧 Squid 默认 80 端口，Endpoint 无需加端口号（非标准端口才需加 `:端口`）。

### 5.2 常用操作

```bash
s3cmd ls                                    # 列出桶
s3cmd mb s3://my-bucket                     # 创建桶
s3cmd put ./file.txt s3://my-bucket/        # 上传
s3cmd ls s3://my-bucket/                    # 列出对象
s3cmd get s3://my-bucket/file.txt ./        # 下载
s3cmd rm s3://my-bucket/file.txt            # 删除
s3cmd rb s3://my-bucket                     # 删桶（需先清空）
s3cmd sync ./dir/ s3://my-bucket/           # 递归同步目录
s3cmd setacl s3://my-bucket/file.txt --acl-public   # 设公开读
# 公开文件浏览器直接访问：http://192.168.12.176/my-bucket/file.txt
```

> ⚠ Ceph RGW 的 ACL 分 **bucket 级 + object 级**两层：桶设公开不会自动应用到对象，每个文件需单独 `--acl-public`。

---

## 六、AWS CLI

```bash
pip install awscli
aws configure          # 输入 AK/SK，region 留空

# 配置端点（写入 ~/.aws/config，后续免 --endpoint-url）
cat >> ~/.aws/config << 'CONF'
[default]
s3 =
    endpoint_url = http://192.168.12.176
CONF

# 常用操作
aws --endpoint-url http://192.168.12.176 s3 ls
aws --endpoint-url http://192.168.12.176 s3 mb s3://aws-bucket
aws --endpoint-url http://192.168.12.176 s3 cp ./test.jpg s3://aws-bucket/
aws --endpoint-url http://192.168.12.176 s3 cp s3://aws-bucket/test.jpg ./
aws --endpoint-url http://192.168.12.176 s3 ls s3://aws-bucket/
aws --endpoint-url http://192.168.12.176 s3 presign s3://aws-bucket/test.jpg --expires-in 3600   # 预签名 URL
```

---

## 七、Python SDK（boto3）

```python
import boto3
from botocore.client import Config

s3 = boto3.client(
    "s3",
    endpoint_url="http://192.168.12.176",
    aws_access_key_id="<你的 access_key>",
    aws_secret_access_key="<你的 secret_key>",
    config=Config(signature_version="s3v4"),
    region_name="",
)

s3.create_bucket(Bucket="my-python-bucket")                    # 创建桶
s3.put_object(Bucket="my-python-bucket", Key="hello.txt", Body=b"Hello RGW!")   # 上传字符串
s3.upload_file("/etc/hosts", "my-python-bucket", "hosts.txt")  # 上传本地文件
s3.download_file("my-python-bucket", "hello.txt", "/tmp/download.txt")          # 下载
for obj in s3.list_objects_v2(Bucket="my-python-bucket").get("Contents", []):
    print(obj["Key"], obj["Size"])                             # 列举
url = s3.generate_presigned_url("get_object",
    Params={"Bucket": "my-python-bucket", "Key": "hello.txt"}, ExpiresIn=3600)  # 预签名
s3.delete_object(Bucket="my-python-bucket", Key="hello.txt")   # 删除
```

> Node.js（aws-sdk）与 Java 的接入要点相同：`endpoint = http://192.168.12.176`、**必须开启 path-style**（`s3ForcePathStyle: true` / `withPathStyleAccessEnabled(true)`）、签名 v4。

---

## 八、高级配置

### 8.1 Nginx 反代 + HTTPS（生产推荐）

```bash
apt install -y nginx
mkdir -p /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/rgw.key -out /etc/nginx/ssl/rgw.crt \
  -subj "/CN=s3.ceph.local"
```

```nginx
# /etc/nginx/sites-available/rgw
server {
    listen 443 ssl;
    server_name s3.ceph.local;
    ssl_certificate /etc/nginx/ssl/rgw.crt;
    ssl_certificate_key /etc/nginx/ssl/rgw.key;

    location / {
        proxy_pass http://127.0.0.1:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

```bash
ln -s /etc/nginx/sites-available/rgw /etc/nginx/sites-enabled/
nginx -s reload
```

### 8.2 配额、生命周期、CORS 速查

> 以下 `radosgw-admin` 命令在 ceph1 上需加 `cephadm shell -- ` 前缀。

```bash
# 桶配额（10GB / 1000 对象）
cephadm shell -- radosgw-admin quota set --uid=testuser --quota-scope=bucket --max-size=10G
cephadm shell -- radosgw-admin quota set --uid=testuser --quota-scope=bucket --max-objects=1000
cephadm shell -- radosgw-admin quota enable --uid=testuser

# 用户配额（该用户所有桶总容量 100GB）
cephadm shell -- radosgw-admin quota set --uid=testuser --quota-scope=user --max-size=100G

# 生命周期：logs/ 前缀对象 30 天后过期
cat > lifecycle.json << 'JSON'
{ "Rules": [ { "ID": "expire-old-logs", "Status": "Enabled",
    "Filter": { "Prefix": "logs/" }, "Expiration": { "Days": 30 } } ] }
JSON
aws --endpoint-url http://192.168.12.176 s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket --lifecycle-configuration file://lifecycle.json

# CORS（Web 前端跨域）
cat > cors.xml << 'XML'
<CORSConfiguration><CORSRule>
    <AllowedOrigin>*</AllowedOrigin>
    <AllowedMethod>GET</AllowedMethod><AllowedMethod>PUT</AllowedMethod><AllowedMethod>POST</AllowedMethod>
    <AllowedHeader>*</AllowedHeader>
</CORSRule></CORSConfiguration>
XML
aws --endpoint-url http://192.168.12.176 s3api put-bucket-cors \
  --bucket my-bucket --cors-configuration file://cors.xml
```

---

## 九、故障排查

| # | 错误信息 | 原因 | 解决 |
|---|---------|------|------|
| 1 | `AccessDenied` | 密钥错误或无权限 | 检查 AK/SK；对象需单独设 public ACL |
| 2 | `SignatureDoesNotMatch` | 客户端时间不同步 | `chronyc tracking` 校正时间 |
| 3 | `BucketAlreadyExists` | 桶名已被占用 | S3 桶名全局唯一，换名 |
| 4 | `EndpointConnectionError` | RGW 端口不可达 | `telnet 192.168.12.176 80` |
| 5 | `NoSuchBucket` | 桶不存在 / endpoint 错误 | 确认桶名与 endpoint URL |
| 6 | `EntityTooLarge` | 单文件超 5GB | 用分片上传（multipart） |
| 7 | `InvalidAccessKeyId` | key 被禁用/删除 | `radosgw-admin user info` 确认 |
| 8 | curl 访问返回 `403` | 需签名认证，不支持匿名 | 用 s3cmd / AWS CLI / SDK（403 属正常响应） |

**诊断流程**：RGW 运行中？(`ceph orch ps --daemon-type rgw`) → 端口通？(`telnet ... 80`) → 时间同步？(`chronyc tracking`) → 认证正确？(`radosgw-admin user info`)。

**看日志**：

```bash
podman logs <rgw容器名> 2>&1 | tail -50
cephadm enter --name rgw.myrgw.ceph1
tail -100 /var/log/ceph/ceph-client.rgw.myrgw.ceph1.log
exit
```

---

## 十、卸载与清理

```bash
# 1. 清空并删除桶
s3cmd del s3://my-bucket/*
s3cmd rb s3://my-bucket
# 或 aws --endpoint-url http://192.168.12.176 s3 rm s3://my-bucket --recursive

# 2. 删除 S3 用户
cephadm shell -- radosgw-admin user rm --uid=testuser

# 3. 删除 RGW 服务
ceph orch rm rgw.myrgw

# 4. 删除 RGW 存储池（确认无数据后）
ceph osd pool delete rgw_data rgw_data --yes-i-really-really-mean-it
ceph osd pool delete rgw_meta rgw_meta --yes-i-really-really-mean-it
ceph osd pool delete rgw_log rgw_log --yes-i-really-really-mean-it
```

---

## 参考信息

| 项目 | 位置 |
|------|------|
| 集群部署文档 | `ceph部署-副本模式.md` |
| CephFS 文件存储指南 | `cephfs-guide.md` |
| 客户端远程挂载（RBD） | `ceph-client-mount.md` |
| 新增节点文档 | `ceph-add-node.md` |
| Ceph 官方 RGW 文档 | https://docs.ceph.com/en/squid/radosgw/ |
| AWS S3 API 参考 | https://docs.aws.amazon.com/AmazonS3/latest/API/Welcome.html |
