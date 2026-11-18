
# OpenOndemand调研手册
# Ubuntu24
> 前置条件:已经安装slurm集群，同时管理节点的/home目录已经共享到计算节点的/home
> 非特殊说明的所有操作，均在管理节点操作
一、 软件安装
```
sudo apt install apt-transport-https ca-certificates
wget -O /tmp/ondemand-release-web_4.2.0-noble_all.deb https://apt.osc.edu/ondemand/4.2/ondemand-release-web_4.2.0-noble_all.deb
sudo apt install /tmp/ondemand-release-web_4.2.0-noble_all.deb
sudo apt update

sudo apt install ondemand
#如果下载太慢，可以考虑去往下载站手动下载安装：https://apt.osc.edu/ondemand/4.2/web/apt/

apt-get install ondemand-dex

sudo systemctl enable ondemand-dex.service
sudo systemctl start ondemand-dex.service

sudo systemctl start apache2
sudo systemctl enable apache2
```
## 二、配置LDAP
```
sudo apt install -y slapd ldap-utils
systemctl enable slapd
systemctl status slapd

#如需重新配置密码
sudo dpkg-reconfigure slapd

#查询本机ldap
ldapsearch -x -LLL -b dc=ps,dc=com -H ldap://127.0.0.1

#创建用户admin。各节点都要添加，且ID要一致
useradd -m -d /home/admin -s /bin/bash admin
```
> 生成密钥:
```
slappasswd -s Puersai.168
```
> 创建modifydb.ldif并修改内容：
```
dn: olcDatabase={1}mdb,cn=config
changetype: modify
replace: olcSuffix
olcSuffix: dc=ps,dc=com
-
replace: olcRootDN
olcRootDN: cn=Manager,dc=ps,dc=com
-
replace: olcRootPW
olcRootPW: 替换生成的密钥
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
> 创建base.ldif并修改内容：
```
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
> 创建用户容器people.ldif：
```
dn: ou=people,dc=ps,dc=com
objectClass: top
objectClass: organizationalUnit
ou: people
```
> 生成用户密码哈希密钥：
```
slappasswd -s 'Puersai.168'
```
> 创建用户users.ldif:
```
dn: uid=<admin>,ou=people,dc=ps,dc=com
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: <admin>
cn: <admin>
sn: <admin>
givenName: <admin>
mail: <admin>@ps.com
uidNumber: <10002>
gidNumber: <10002>
homeDirectory: /home/admin
loginShell: /bin/bash
userPassword: 替换生成的密钥
```
> 执行：
```
sudo ldapmodify -Y EXTERNAL -H ldapi:/// -f modifydb.ldif

ldapadd -x -H ldap://127.0.0.1 -D "cn=Manager,dc=ps,dc=com" -w 'Puersai.168' -f base.ldif

ldapadd -x -H ldap://127.0.0.1 -D "cn=Manager,dc=ps,dc=com" -w 'Puersai.168' -f people.ldif

ldapadd -x -H ldap://127.0.0.1 -D "cn=Manager,dc=ps,dc=com" -w 'Puersai.168' -f users.ldif

ldapsearch -x -LLL -H ldap://127.0.0.1 -b dc=ps,dc=com -D "cn=Manager,dc=ps,dc=com" -w 'Puersai.168'
```
## 三、配置ondemand-dex
> 创建本地ssl证书
```
mkdir -p /etc/ood/ssl
openssl req -x509 -nodes -days 3650 -newkey rsa:2048 \
  -keyout /etc/ood/ssl/ood.key \
  -out /etc/ood/ssl/ood.crt \
  -subj "/CN=<ip>" \
  -addext "subjectAltName=IP:<ip>"
```
> 修改ondemand-dex配置文件：/etc/ood/config/ood_portal.yml
```
---
servername: <ip>
port: 443
dex_uri: /dex
ssl:
  - 'SSLCertificateFile "/etc/ood/ssl/ood.crt"'
  - 'SSLCertificateKeyFile "/etc/ood/ssl/ood.key"'
oidc_provider_metadata_url: http://localhost:5556/dex/.well-known/openid-configuration
oidc_settings:
  OIDCSSLValidateServer: Off
  OIDCOAuthSSLValidateServer: Off
dex:
  connectors:
    - type: ldap
      id: ldap
      name: LDAP
      config:
        host: <ip>:389
        insecureNoSSL: true
        insecureSkipVerify: true
        bindDN: cn=Manager,dc=ps,dc=com
        bindPW: Puersai.168
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
> 更新apache2服务
```
/opt/ood/ood-portal-generator/sbin/update_ood_portal

#apache2里面开启ssl
a2enmod ssl

systemctl restart apache2 ondemand-dex
```
> 访问测试：http://192.168.12.178
## 四、配置slurm集群
> 编辑文件/etc/ood/config/clusters.d/<clustername>.yml
```
---
v2:
   metadata:
     title: "Puersai HPC"
   login:
     host: <主机ip>
   job:
     adapter: "slurm"
     cluster: "<集群名>"
     bin: "/usr/bin"
     conf: "/etc/slurm/slurm.conf"
     # bin_overrides:
       # sbatch: "/usr/local/bin/sbatch"
       # squeue: ""
       # scontrol: ""
       # scancel: ""
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
         # Add the TurboVNC installation directory to the PATH (note: this may be different on your system)
         export PATH="/opt/TurboVNC/bin:$PATH"
         export WEBSOCKIFY_CMD="/usr/local/bin/websockify"
         %s
       set_host: "host=$(hostname -A | awk '{print $1}')"
```
## 五、配置交互式应用
> 安装软件依赖
```
#计算节点执行

#ncat：下载地址：https://nmap.org/download.html

wget https://nmap.org/dist/ncat-7.99-1.x86_64.rpm

apt-get install alien

alien ncat-7.99-1.x86_64.rpm

dpkg --install ncat_7.99-2_amd64.deb

#检查
ncat --version

--------------------------------------------------------------------------------

#TurboVNC
wget -q -O- https://packagecloud.io/dcommander/turbovnc/gpgkey | gpg --dearmor >/etc/apt/trusted.gpg.d/TurboVNC.gpg

echo "deb [signed-by=/etc/apt/trusted.gpg.d/TurboVNC.gpg] https://packagecloud.io/dcommander/turbovnc/any/ any main" > /etc/apt/sources.list.d/turbovnc.list

apt update

apt install turbovnc

--------------------------------------------------------------------------------

#websockify （>0.13.0）下载地址：https://github.com/novnc/websockify

apt install python3-setuptools

#解压并进入目录执行：
python3 setup.py install

#检查
websockify --help
```
> 验证是否正常工作
```
#计算节点执行：

nc -l 5432

#浏览器打开：
https://<ip>/node/ps/5432

#返回计算节点，检查输出如下：
GET /node/ps/5432 HTTP/1.1
Host: 192.168.12.178
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:152.0) Gecko/20100101 Firefox/152.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.9,zh-TW;q=0.8,zh-HK;q=0.7,en-US;q=0.6,en;q=0.5
Accept-Encoding: gzip, deflate, br, zstd
Upgrade-Insecure-Requests: 1
Sec-Fetch-Dest: document
Sec-Fetch-Mode: navigate
Sec-Fetch-Site: none
Sec-Fetch-User: ?1
Priority: u=0, i
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
## 六、开启VNC
> 必须配置nfs，管理节点把/home目录共享到计算节点/home目录
```
#计算节点执行
sudo apt install -y mate-desktop-environment-core xfce4 \
  gnome-session-flashback metacity feh kde-plasma-desktop git
```
```
#管理节点执行
mkdir -p /etc/ood/config/apps/bc_desktop

#创建/etc/ood/config/apps/bc_desktop/<集群名>.yml：
---
title: "Desktop"
cluster: <mycluster>

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
> 修改/var/www/ood/apps/sys/bc_desktop/template/script.sh.erb:
```
#管理节点执行
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
## 七、添加其他应用
> jupyter
## 七、品牌化
> 更新
```
/opt/ood/nginx_stage/sbin/nginx_stage nginx --user admin -s stop ; /opt/ood/nginx_stage/sbin/nginx_stage pun --user admin
```
一、 VNC无法连接会话问题
> 原因分析：openondemand的VNC启动依赖于自带的桌面启动脚本，而该脚本是针对于老版本的系统做的适配，因此在新系统上可能出现无法正常启动vnc的情况。因此需要更改启动脚本
> ubuntu24：
### 客户端安装：
```
sudo apt install -y mate-desktop-environment-core xfce4 \
  gnome-session-flashback metacity feh kde-plasma-desktop
```
### /var/www/ood/apps/sys/bc_desktop/template/script.sh.erb:
```
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
## ondemand-dex报错：unable to open database file: no such file or directory
```
rm -f /etc/ood/dex/dex.db
chown ondemand-dex:ondemand-dex /etc/ood/dex
chmod 700 /etc/ood/dex
systemctl restart ondemand-dex
```