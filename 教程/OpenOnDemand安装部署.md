# Open OnDemand 安装部署手册（Ubuntu 24.04）

> **适用环境**：Ubuntu 24.04 (Noble) + Slurm 集群
> **前置条件**：已完成 Slurm 集群的安装与配置（控制节点 / 计算节点均可正常调度）
> **文档性质**：调研 + 实操记录，命令均经过实际环境验证

## 环境信息

| 项目 | 本文档示例值 | 说明 |
|------|-------------|------|
| 操作系统 | Ubuntu 24.04 (Noble) | OOD 主节点与计算节点 |
| Open OnDemand | 4.2.0 | 官方 apt 源 `apt.osc.edu` |
| OOD 主节点 IP | `192.168.12.10` | 下文所有 `<ip>` 均以此为例，实际部署请替换 |
| Slurm 集群名 | `mycluster` | 与 `slurm.conf` 中 `ClusterName` 一致 |
| LDAP 后缀 | `dc=ps,dc=com` | |
| LDAP 管理员 | `cn=Manager,dc=ps,dc=com` | |
| 测试用户 | `admin` | |

> **安全提示**：文中出现的密码、SSHA 哈希均为示例环境值，实际部署时务必重新生成并替换，切勿将真实密码提交到文档或版本库。

## 目录

- [一、软件安装](#一软件安装)
- [二、配置 LDAP](#二配置-ldap)
- [三、配置 OOD 门户与 Dex 认证（LDAP）](#三配置-ood-门户与-dex-认证ldap)
- [四、配置 Slurm 集群](#四配置-slurm-集群)
- [五、配置交互式应用依赖](#五配置交互式应用依赖)
- [六、连通性验证](#六连通性验证)
- [七、开启 VNC 桌面（bc_desktop）](#七开启-vnc-桌面bc_desktop)
- [八、品牌化](#八品牌化)
- [九、常用运维命令](#九常用运维命令)
- [十、故障排查](#十故障排查)
- [附录：端口清单与参考链接](#附录端口清单与参考链接)

---

## 一、软件安装

在 OOD 主节点（Web 门户所在节点）执行：

```bash
# 安装 apt 传输依赖
sudo apt install apt-transport-https ca-certificates -y

# 下载并安装 OOD 官方 apt 源（4.2.0 noble 版）
wget -O /tmp/ondemand-release-web_4.2.0-noble_all.deb \
  https://apt.osc.edu/ondemand/4.2/ondemand-release-web_4.2.0-noble_all.deb
sudo apt install /tmp/ondemand-release-web_4.2.0-noble_all.deb
sudo apt update

# 安装 ONDemand 主程序与 Dex 认证组件
sudo apt install ondemand -y
sudo apt install ondemand-dex -y
```

> 如果下载太慢，可以去官方下载站手动下载 deb 包后本地安装：<https://apt.osc.edu/ondemand/4.2/web/apt/>

设置开机自启并启动服务：

```bash
# Dex 认证服务
sudo systemctl enable ondemand-dex.service
sudo systemctl start ondemand-dex.service

# Apache2（OOD 门户基于 Apache 运行）
sudo systemctl start apache2
sudo systemctl enable apache2
```

安装完成后确认状态：

```bash
systemctl status apache2 ondemand-dex
```

## 二、配置 LDAP

在 OOD 主节点安装 OpenLDAP 服务端（slapd）与客户端工具：

```bash
sudo apt install -y slapd ldap-utils
systemctl enable slapd
systemctl status slapd

# 如需重新配置（域、管理员密码等）
sudo dpkg-reconfigure slapd

# 查询本机 LDAP 当前内容
ldapsearch -x -LLL -b dc=ps,dc=com -H ldap://127.0.0.1
```

### 2.1 创建系统用户（测试账号）

```bash
useradd -m -d /home/admin -s /bin/bash admin
```

### 2.2 修改 LDAP 数据库配置（modifydb.ldif）

先生成管理员密码哈希：

```bash
slappasswd -s Puersai.168
# 输出形如：{SSHA}SZ6jajXKcYG5dh2vRVUo7u5xNlsyd+U3（每次生成结果不同，属正常现象）
```

创建 `modifydb.ldif`，将 `<密钥>` 替换为上一步的输出：

```ldif
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ps,dc=com
-
replace: olcRootDN
olcRootDN: cn=Manager,dc=ps,dc=com
-
replace: olcRootPW
olcRootPW: {SSHA}SZ6jajXKcYG5dh2vRVUo7u5xNlsyd+U3
-
replace: olcDbMaxSize
olcDbMaxSize: 1073741824
-
replace: olcDbDirectory
olcDbDirectory: /var/lib/ldap
-
replace: olcDbIndex
olcDbIndex: objectClass eq
```

### 2.3 创建目录结构（base.ldif）

```ldif
dn: dc=ps,dc=com
objectClass: top
objectClass: dcObject
objectClass: organization
o: ps
dc: ps

dn: cn=Manager,dc=ps,dc=com
objectClass: organizationalRole
cn: Manager
description: LDAP administrator
```

### 2.4 创建用户容器（people.ldif）

```ldif
dn: ou=people,dc=ps,dc=com
objectClass: top
objectClass: organizationalUnit
ou: people
```

### 2.5 创建 LDAP 用户（users.ldif）

先生成用户密码哈希：

```bash
slappasswd -s 'Puersai.168'
```

创建 `users.ldif`（`<admin>`、`<10002>` 等按实际规划替换）：

```ldif
dn: uid=admin,ou=people,dc=ps,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: admin
cn: admin
sn: admin
givenName: admin
mail: admin@ps.com
uidNumber: 10002
gidNumber: 10002
homeDirectory: /home/admin
loginShell: /bin/bash
userPassword: {SSHA}BBy/DLZIwdyvMkw8IQq/6wQz62XOWDvv
```

### 2.6 执行导入并验证

```bash
# 1. 修改数据库配置（注意 -Y EXTERNAL 走本地 socket）
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f modifydb.ldif

# 2. 导入目录结构
ldapadd -x -H ldap://127.0.0.1 -D "cn=Manager,dc=ps,dc=com" -w 'Puersai.168' -f base.ldif

# 3. 导入用户
ldapadd -x -H ldap://127.0.0.1 -D "cn=Manager,dc=ps,dc=com" -w 'Puersai.168' -f users.ldif

# 4. 验证查询
ldapsearch -x -LLL -H ldap://127.0.0.1 -b dc=ps,dc=com \
  -D "cn=Manager,dc=ps,dc=com" -w 'Puersai.168'
```

> **注意**：`-D` 前的 `-x` 表示简单认证（密码方式），不要与 `-Y EXTERNAL` 混淆；批量建用户时复制 `users.ldif` 条目并修改 `uid` / `uidNumber` / `gidNumber` 即可。

## 三、配置 OOD 门户与 Dex 认证（LDAP）

### 3.1 生成本地 SSL 证书

```bash
mkdir -p /etc/ood/ssl
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/ood/ssl/ood.key \
  -out /etc/ood/ssl/ood.crt \
  -subj "/CN=192.168.12.10" \
  -addext "subjectAltName=IP:192.168.12.10"
```

> 自签名证书浏览器会告警，生产环境建议替换为机构 CA 签发证书；`CN` 与 `SAN` 需与实际访问地址一致。

### 3.2 修改门户配置 /etc/ood/config/ood_portal.yml

完整配置（`dex` 段将 LDAP 作为认证源，OIDC 走 Dex）：

```yaml
---
servername: 192.168.12.10
port: 443
dex_uri: /dex
ssl:
  - 'SSLCertificateFile "/etc/ood/ssl/ood.crt"'
  - 'SSLCertificateKeyFile "/etc/ood/ssl/ood.key"'

# OIDC 由本机 Dex 提供元数据（5556 为 Dex 默认监听端口）
oidc_provider_metadata_url: http://localhost:5556/dex/.well-known/openid-configuration
oidc_settings:
  OIDCSSLValidateServer: Off
  OIDCOAuthSSLValidateServer: Off

# OIDC 会话加密密钥，建议 32 位以上随机字符串
oidc_crypto_passphrase: "MySuperSecretPassphrase1234567890"

# Dex 认证器：对接上文 LDAP
dex:
  connectors:
    - type: ldap
      id: ldap
      name: LDAP
      config:
        host: 192.168.12.10:389
        insecureNoSSL: true        # 内网明文连接；生产环境建议启用 LDAPS
        insecureSkipVerify: true
        bindDN: cn=Manager,dc=ps,dc=com
        bindPW: Puersai.168        # LDAP 管理员密码
        userSearch:
          baseDN: ou=people,dc=ps,dc=com
          filter: "(objectClass=posixAccount)"
          username: uid
          idAttr: uid
          emailAttr: mail
          nameAttr: cn
          preferredUsernameAttr: uid

host_regex: "[^/]+"
node_uri: '/node'
rnode_uri: '/rnode'
```

### 3.3 应用配置并重启服务

```bash
# 由 ood_portal.yml 重新生成 Apache 与 Dex 实际配置
/opt/ood/ood-portal-generator/sbin/update_ood_portal

# 启用 SSL 模块（首次配置时执行一次即可）
sudo a2enmod ssl

# 重启生效
systemctl restart apache2 ondemand-dex
```

### 3.4 访问测试

浏览器打开 `https://192.168.12.10`（注意是 **https**），应跳转到 Dex 登录页，使用 LDAP 用户（如 `admin`）登录后进入 OOD 仪表盘。

## 四、配置 Slurm 集群

编辑 `/etc/ood/config/clusters.d/mycluster.yml`（文件名即集群名，需带 `.yml` 后缀）：

```yaml
---
v2:
  metadata:
    title: "Puersai HPC"
  login:
    host: 192.168.12.10          # 登录节点地址
  job:
    adapter: "slurm"
    cluster: "mycluster"         # 与 slurm.conf 的 ClusterName 一致
    bin: "/usr/bin"
    conf: "/etc/slurm/slurm.conf"
    # 如命令路径特殊可用 bin_overrides：
    # bin_overrides:
    #   sbatch: "/usr/local/bin/sbatch"
    #   squeue: ""
    #   scontrol: ""
    #   scancel: ""
    copy_environment: false
  batch_connect:
    basic:
      script_wrapper: |
        module purge
        %s
      set_host: "host=$(hostname -A | awk '{print $1}')"
    vnc:
      script_wrapper: |
        module purge
        # 将 TurboVNC 安装目录加入 PATH（路径以实际安装为准）
        export PATH="/opt/TurboVNC/bin:$PATH"
        export WEBSOCKIFY_CMD="/usr/local/bin/websockify"
        %s
      set_host: "host=$(hostname -A | awk '{print $1}')"
```

修改后无需重启 Apache，刷新页面即可看到新集群；作业提交失败时可先在该节点手工执行 `sbatch --version` 确认 Slurm 客户端可用。

## 五、配置交互式应用依赖

以下依赖服务于 VNC 交互式桌面（bc_desktop），在**计算节点**上安装：

### 5.1 ncat（端口连通性测试用）

```bash
# 方式一（推荐）：Ubuntu 官方源直接安装
apt install ncat -y

# 方式二：RPM 包转换安装（官方下载地址 https://nmap.org/download.html）
apt-get install alien -y
wget https://nmap.org/dist/ncat-7.99-1.x86_64.rpm
alien ncat-7.99-1.x86_64.rpm          # 转换生成 ncat_7.99-2_amd64.deb
dpkg --install ncat_7.99-2_amd64.deb  # 安装转换后的 deb 包

# 检查
ncat --version
```

### 5.2 TurboVNC

```bash
wget -q -O- https://packagecloud.io/dcommander/turbovnc/gpgkey \
  | gpg --dearmor >/etc/apt/trusted.gpg.d/TurboVNC.gpg

echo "deb [signed-by=/etc/apt/trusted.gpg.d/TurboVNC.gpg] https://packagecloud.io/dcommander/turbovnc/any/ any main" \
  > /etc/apt/sources.list.d/turbovnc.list

apt update
apt install turbovnc -y
```

### 5.3 websockify（需 > 0.13.0）

源码地址：<https://github.com/novnc/websockify>

```bash
apt install python3-setuptools

# 解压源码包并进入目录后执行：
python3 setup.py install

# 检查
websockify --help
```

## 六、连通性验证

验证 OOD 的 `/node` 反向代理链路是否打通：

```bash
# 1. 计算节点上监听测试端口
nc -l 5432
```

```text
# 2. 浏览器打开（host 为计算节点主机名，端口为监听端口）：
https://192.168.12.10/node/ps/5432
```

```text
# 3. 返回计算节点，若看到类似以下 HTTP 请求头，说明 Apache → 计算节点链路已打通
# （重点关注 X-Forwarded-User 与 OIDC_CLAIM_*，表明认证信息已正确传递）：
GET /node/ps/5432 HTTP/1.1
Host: 192.168.12.178
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:152.0) Gecko/20100101 Firefox/152.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
OIDC_CLAIM_at_hash: GPlr2eqC9iMYd7dRGpUbxg
OIDC_CLAIM_c_hash: wbCBZEhpzS0DKqMkv_i_Aw
OIDC_CLAIM_name: admin
X-Forwarded-Proto: https
X-Forwarded-User: admin
X-Forwarded-Escaped-Uri: %2fnode%2fps%2f5432
Cookie: mod_auth_openidc_session=dd0860c72f9ebdf749849fb9da12e1c6678b3b78
X-Forwarded-For: 192.168.12.1
X-Forwarded-Host: 192.168.12.178
X-Forwarded-Server: 192.168.12.178
```

## 七、开启 VNC 桌面（bc_desktop）

### 7.1 计算节点安装桌面环境

```bash
sudo apt install -y mate-desktop-environment-core xfce4 \
  gnome-session-flashback metacity feh kde-plasma-desktop
```

> 桌面环境较大，可按需选择安装（至少一种）；GNOME 桌面需 `gnome-session-flashback`（普通 GNOME 在批处理作业中无法启动）。

### 7.2 创建应用配置

```bash
mkdir -p /etc/ood/config/apps/bc_desktop
```

创建 `/etc/ood/config/apps/bc_desktop/mycluster.yml`：

```yaml
---
title: "Desktop"
cluster: mycluster

attributes:
  desktop:
    label: "Desktop Environment"
    widget: select
    value: kde
    options:
      - "kde"
      - "gnome"
      - "mate"
      - "xfce"
  bc_vnc_idle: 0
  bc_vnc_resolution:
    required: true
  node_type: null

form:
  - bc_vnc_idle
  - desktop
  - bc_account
  - bc_num_hours
  - bc_num_slots
  - node_type
  - bc_queue
  - bc_vnc_resolution
  - bc_email_on_started
```

### 7.3 修改桌面启动脚本

Ubuntu 24.04 上官方自带的桌面启动脚本基于老版本系统适配，直接使用会出现 VNC 无法连接会话的问题，需将 `/var/www/ood/apps/sys/bc_desktop/template/script.sh.erb` 替换为以下内容（完整脚本，适配 mate / xfce / gnome / kde 四种桌面）：

```bash
#!/usr/bin/env bash

# Change working directory to user's home directory
cd "${HOME}"

# Reset module environment when Lmod/Environment Modules is available
if [[ $(type -t module) == "function" ]]; then
  module purge && module restore
fi

# Ensure that the user's configured login shell is used
export SHELL="$(getent passwd $USER | cut -d: -f7)"

# Avoid conda/user dbus-launch taking precedence over system binaries
# https://github.com/OSC/ondemand/issues/700
SAFE_PATH=$(echo -n "$PATH" | tr : '\0' | grep -zvE ^"$HOME"'($|/)' | tr '\0' : | head -c -1)

# Slurm batch jobs have no pam_systemd login session
unset SESSION_MANAGER
unset WAYLAND_DISPLAY
unset GNOME_KEYRING_CONTROL
unset SSH_AUTH_SOCK
unset DBUS_SESSION_BUS_ADDRESS
export GDK_BACKEND=x11
export XDG_SESSION_TYPE=x11
export NO_AT_BRIDGE=1
export XDG_RUNTIME_DIR="${TMPDIR:-/tmp}/ood-vnc-${USER}-$$"
mkdir -p "${XDG_RUNTIME_DIR}"
chmod 700 "${XDG_RUNTIME_DIR}"

# Stub systemd-run (GNOME/KDE may call it; no user systemd in batch jobs)
STUB_BIN="${TMPDIR:-/tmp}/ood-bin-${USER}-$$"
mkdir -p "${STUB_BIN}"
cat > "${STUB_BIN}/systemd-run" << 'STUBEOF'
#!/bin/bash
while [[ $# -gt 0 ]]; do
  case "$1" in
    --user|--scope|--collect|--quiet|--property=*|--service-type=|--same-dir) shift ;;
    --) shift; break ;;
    *) break ;;
  esac
done
[[ $# -gt 0 ]] && exec "$@"
exec "${SHELL:-/bin/bash}"
STUBEOF
chmod +x "${STUB_BIN}/systemd-run"
export PATH="${STUB_BIN}:${SAFE_PATH}"

# Disable autostart services that crash in headless batch jobs
AUTOSTART="${HOME}/.config/autostart"
rm -rf "${AUTOSTART}"
mkdir -p "${AUTOSTART}"
for service in \
  light-locker xiccd nm-applet vmware-user \
  polkit-gnome-authentication-agent-1 polkit-mate-authentication-agent-1 \
  gnome-keyring-gpg gnome-keyring-pkcs11 gnome-keyring-secrets gnome-keyring-ssh \
  pulseaudio tracker-miner-fs-3 tracker-extract tracker-miner-apps tracker-miner-user-guides \
  mate-power-manager xfce4-power-manager update-notifier at-spi-dbus-bus \
  gdu-notification-daemon bluedevil kalendarac kdeconnectd discover-notifier; do
  cat > "${AUTOSTART}/${service}.desktop" << EOF
[Desktop Entry]
Type=Application
Name=Disabled
Hidden=true
EOF
done

# Start desktop inside D-Bus session (all desktop types handled here)
echo "Launching desktop '<%= context.desktop %>'..."

case '<%= context.desktop %>' in
  mate)
    dbus-run-session -- mate-session
    ;;
  xfce)
    dbus-run-session -- xfce4-session
    ;;
  gnome)
    export XDG_CURRENT_DESKTOP="GNOME-Flashback:GNOME"
    export XDG_MENU_PREFIX="gnome-flashback-"
    export GNOME_SHELL_SESSION_MODE=gnome-flashback-metacity
    export GNOME_SESSION_MODE=gnome-flashback-metacity
    export XDG_SESSION_CLASS=user
    export XDG_SESSION_DESKTOP=gnome-flashback-metacity

    dbus-run-session -- bash -c '
      apply_gnome_wallpaper() {
        command -v feh >/dev/null 2>&1 || return 0
        local uri opts path
        uri=$(gsettings get org.gnome.desktop.background picture-uri 2>/dev/null | tr -d "\047")
        opts=$(gsettings get org.gnome.desktop.background picture-options 2>/dev/null | tr -d "\047")
        [[ -z "$uri" || "$uri" == "" ]] && return 0
        if [[ "$uri" == file://* ]]; then
          path="${uri#file://}"
        else
          return 0
        fi
        [[ -f "$path" ]] || return 0

        # GNOME "wallpaper" means tile; VNC desktops should always fill the screen
        if [[ "$opts" == "wallpaper" ]]; then
          gsettings set org.gnome.desktop.background picture-options zoom 2>/dev/null || true
          opts=zoom
        fi

        case "$opts" in
          centered)  feh --bg-center "$path" ;;
          scaled)    feh --bg-scale "$path" ;;
          spanned)   feh --bg-max "$path" ;;
          stretched|zoom|*) feh --bg-fill "$path" ;;
        esac
      }

      # GNOME Flashback has no background plugin; default XML/SVG wallpapers render black in VNC
      WALLPAPER=""
      for w in /usr/share/backgrounds/warty-final-ubuntu.png /usr/share/backgrounds/*.jpg /usr/share/backgrounds/*.png; do
        [[ -f "$w" ]] && WALLPAPER="$w" && break
      done
      if [[ -n "$WALLPAPER" ]]; then
        gsettings set org.gnome.desktop.background picture-uri "file://${WALLPAPER}" 2>/dev/null || true
        gsettings set org.gnome.desktop.background picture-options zoom 2>/dev/null || true
      fi

      # feh paints wallpaper and reacts to Settings -> Background changes
      (
        sleep 3
        apply_gnome_wallpaper
        gsettings monitor org.gnome.desktop.background 2>/dev/null | while read -r _; do
          sleep 0.3
          apply_gnome_wallpaper
        done
      ) &

      exec /usr/libexec/gnome-flashback-metacity
    '
    ;;
  kde)
    export XDG_CURRENT_DESKTOP=KDE
    export DESKTOP_SESSION=plasma
    export KDE_FULL_SESSION=true
    if command -v startplasma-x11 >/dev/null 2>&1; then
      dbus-run-session -- startplasma-x11
    elif command -v startkde >/dev/null 2>&1; then
      dbus-run-session -- startkde
    else
      echo "Error: install KDE Plasma on compute nodes (apt install kde-plasma-desktop)" >&2
      exit 1
    fi
    ;;
  *)
    echo "Unsupported desktop: <%= context.desktop %>" >&2
    exit 1
    ;;
esac

echo "Desktop '<%= context.desktop %>' ended with $? status..."
```

保存后刷新 OOD 页面，在 **Interactive Apps → Desktop** 提交作业即可获得 VNC 桌面会话。

## 八、品牌化

### 8.1 替换 Logo 与网站图标

准备品牌资源文件（logo、暗色 logo、favicon），在 `/etc/ood/config/ood_portal.yml` 中追加：

```yaml
brand_assets:
  logo: "/path/to/logo.png"            # 侧边栏顶部 logo（建议透明底 PNG，高约 40px）
  logo_dark: "/path/to/logo-dark.png"  # 暗色主题 logo（可选）
  favicon: "/path/to/favicon.ico"      # 浏览器标签图标
```

应用并重启：

```bash
/opt/ood/ood-portal-generator/sbin/update_ood_portal
systemctl restart apache2
```

资源会被复制到 `/var/www/ood/public/brand_assets/` 下生效。

### 8.2 首页公告

创建 `/etc/ood/config/announcement.yml`：

```yaml
- type: info        # 可选：info / warn / error
  content: "系统将于本周六 22:00-24:00 进行维护，请提前保存作业。"
```

### 8.3 重启用户 Web 服务

修改应用级配置（如 bc_desktop）后，需要重启对应用户的 PUN（Per-User Nginx）才能生效：

```bash
/opt/ood/nginx_stage/sbin/nginx_stage nginx --user admin -s stop ; \
/opt/ood/nginx_stage/sbin/nginx_stage pun --user admin
```

> 将 `admin` 替换为目标用户名；所有已登录用户在配置变更后都需重启各自的 PUN。

## 九、常用运维命令

```bash
# 服务状态
systemctl status apache2 ondemand-dex

# 服务重启
systemctl restart apache2
systemctl restart ondemand-dex

# 门户配置变更（修改 ood_portal.yml 后必须执行）
/opt/ood/ood-portal-generator/sbin/update_ood_portal
systemctl restart apache2 ondemand-dex

# PUN 管理（列出 / 启动 / 停止指定用户）
/opt/ood/nginx_stage/sbin/nginx_stage pun list
/opt/ood/nginx_stage/sbin/nginx_stage pun --user <user>
/opt/ood/nginx_stage/sbin/nginx_stage pun --user <user> -s stop

# 日志位置
#   Apache：        /var/log/apache2/
#   用户 PUN：      /var/log/ondemand-nginx/<user>/
#   Dex 认证：      journalctl -u ondemand-dex
#   作业脚本输出：  OOD 作业页 →  session 日志
```

## 十、故障排查

### 10.1 VNC 无法连接会话问题

**现象**：交互式 Desktop 作业正常启动，但 VNC 连接失败或黑屏。

**原因分析**：Open OnDemand 的 VNC 启动依赖自带的桌面启动脚本（`script.sh.erb`），该脚本针对老版本系统适配，在新系统（如 Ubuntu 24.04）上可能出现无法正常启动 VNC 的情况，需要替换启动脚本。

**解决方法**：

1. 计算节点安装桌面环境（见 [7.1](#71-计算节点安装桌面环境)）；
2. 将 `script.sh.erb` 替换为 [7.3](#73-修改桌面启动脚本) 中的适配脚本；
3. 重启用户 PUN（见 [8.3](#83-重启用户-web-服务)）后重新提交作业。

### 10.2 ondemand-dex 报错：unable to open database file: no such file or directory

```bash
rm -f /etc/ood/dex/dex.db
chown ondemand-dex:ondemand-dex /etc/ood/dex
chmod 700 /etc/ood/dex
systemctl restart ondemand-dex
```

### 10.3 排查思路速查

| 现象 | 检查点 |
|------|--------|
| 页面 502 / 无法登录 | `systemctl status ondemand-dex apache2`；`journalctl -u ondemand-dex` |
| 登录后作业列表报错 | 检查 `clusters.d/*.yml`；主节点能否执行 `squeue` |
| 交互式作业启动失败 | 查看作业 session 日志；确认计算节点已装 TurboVNC / websockify / ncat |
| 桌面黑屏 | 见 10.1；确认所选桌面环境已安装在计算节点 |
| Dex 登录密码被拒 | `ldapsearch` 手工验证用户；检查 Dex 配置中 `bindDN/bindPW` 与 `baseDN/filter` |

## 附录：端口清单与参考链接

**端口清单**

| 端口 | 服务 | 说明 |
|------|------|------|
| 443 | Apache2 | OOD 门户（HTTPS） |
| 5556 | ondemand-dex | Dex OIDC 认证（仅本机回环） |
| 389 | slapd | LDAP（内网明文，生产建议 636 LDAPS） |
| 5900+ | TurboVNC | 计算节点 VNC 会话 |
| 8080+ | websockify | noVNC WebSocket 转发 |

**防火墙放行**（如启用 ufw）：

```bash
sudo ufw allow 443/tcp
```

**参考链接**

- Open OnDemand 官方文档：<https://osc.github.io/ood-documentation/latest/>
- 源码仓库：<https://github.com/OSC/ondemand>
- apt 下载站：<https://apt.osc.edu/ondemand/4.2/web/apt/>
- TurboVNC：<https://packagecloud.io/dcommander/turbovnc>
- websockify：<https://github.com/novnc/websockify>
- nmap/ncat：<https://nmap.org/download.html>
