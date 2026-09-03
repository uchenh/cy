# Ubuntu 密码破解教程

> 适用版本：Ubuntu 22.04 / 24.04 / 26.04（及较新的 LTS）

---

## 0. 使用前提与授权声明（务必先读）

本教程包含两类操作：

- **直接重置密码**：要求你**已经能进入系统、拥有 root/sudo，或有该物理机的访问权限**（纯运维、遗忘密码找回场景）。
- **离线口令破解**：要求你持有某个用户口令哈希的合法副本（例如**自己实验室**、**授权渗透测试**、**CTF 靶机**）。

> ⚠️ 在**未授权**的情况下对他人系统实施上述任何操作均属违法。只能在**本人拥有**或**明确书面授权管理**的机器上使用。[RecoveryMode](https://wiki.ubuntu.com/RecoveryMode)[LostPassword](https://help.ubuntu.com/community/LostPassword)

---

## 1. 原理基础：`/etc/shadow` 的密码哈希

Linux 用户密码**不存储明文**，而是存于 `/etc/shadow` 文件中，每个用户一行的第三个冒号字段即口令哈希，格式为：

```
$ID$<salt>$<hash>
```

各 `$ID` 前缀含义（Ubuntu 各版本差异很大）：

| 前缀 | 算法 | Ubuntu 使用情况 | hashcat mode |
|---|---|---|---|
| `$1$` | MD5-crypt | 旧系统 | `500` |
| `$5$` | SHA-256-crypt | 较旧 | `7400` |
| `$6$` | SHA-512-crypt | **22.04 之前默认** | `1800` |
| `$y$` | **yescrypt** | **Ubuntu 22.04 LTS 起默认** | 视安装而定（见下文） |

示例哈希：
```
$6$rounds=5000$saltsalt$P5i...hash
$y$j9T$uxVFACnNnGBakt9MLrpFf0$SmbSZAge5oa1BfHPBxYGq3mITgHeO/iG2Mdfgo93UN0
```
哈希里只有加了盐的对撞验证结果，无法逆推回明文；「破解」的本质是对候选口令逐一加盐计算哈希并比对。[crypt.5](https://manpages.ubuntu.com/manpages/focal/man5/crypt.5.html)[password-hashing](https://documentation.ubuntu.com/security/docs/security-forms/cryptography/password-hashing/)

> 💡 `*` / `!` 表示该用户**无法通过密码登录**（锁定），无哈希可破。

---

# 场景 A：直接重置密码（有物理访问 / root 权限）

适用：**忘记了本人机器密码**、或需要替授权用户找回。三种方式按推荐度排列。

## A1. GRUB Recovery Mode（最推荐）

Ubuntu 自带恢复模式，UI 操作，无需记忆命令。

1. **重启**，在开机瞬间 UEFI 反复按 `Esc`，或 BIOS 环境长按 `Shift`，进入 GRUB 菜单。
2. 选择 **“Advanced options for Ubuntu”**。
3. 选择带 **`(recovery mode)`** 后缀的内核条目。
4. 在 recovery 菜单里选择 **`root  — Drop to root shell prompt`**。
5. 根分区默认**只读**，先重新挂载为读写：
   ```bash
   mount -o remount,rw /
   ```
6. 若 `/home`、`/boot`、`/tmp` 有独立分区，可挂载全部：
   ```bash
   mount --all
   ```
7. 重置目标用户密码（会要求键入两次新密码）：
   ```bash
   passwd <用户名>
   ```
   或**直接清空密码**（下次登录空密码，可随后再设置）：
   ```bash
   passwd -d <用户名>
   ```
8. 重启生效：
   ```bash
   reboot
   ```

> ⚠️ recovery 菜单中**不建议勾选 “Enable networking”**，部分资料提示可能挂起。[RecoveryMode](https://wiki.ubuntu.com/RecoveryMode)

[A1 ↗](https://documentation.ubuntu.com/authd/edge-docs/howto/enter-recovery-mode/)[LostPassword](https://help.ubuntu.com/community/LostPassword)

## A2. 编辑 GRUB 参数：`init=/bin/bash`（recovery 不可用时的兜底）

跳过 recovery 菜单链，直接以 root 运行 shell。

1. 重启进入 GRUB 菜单，选中默认 Ubuntu 启动项，按 `e` 编辑。
2. 找到以 `linux` 开头的行，把 `ro` 改成 `rw`，并在**行尾追加**：
   ```
   init=/bin/bash
   ```
3. 按 `Ctrl+X`（或 `F10`）启动，直接进入 root shell。
4. 重置密码：
   ```bash
   passwd <用户名>
   ```
5. 重启：
   ```bash
   reboot
   ```

> 💡 注意：`init=/bin/bash` 不经过 systemd 服务链。若需保留 systemd 又进单用户维护，可改用 `systemd.unit=rescue.target`（多为只读、用于修配置）。[Veeam](https://www.veeam.com/blog/ubuntu-linux-defense-secure-boot-single-user.html)[systemd](https://systemd.io/DEBUGGING/)

## A3. Live USB / chroot 重置

系统完全进不去或没有 recovery 时使用。

1. 用 Ubuntu **Live USB** 启动并打开终端。
2. 确认根分区设备号：
   ```bash
   lsblk        # 或 sudo fdisk -l
   ```
3. 挂载根分区并 chroot：
   ```bash
   sudo mount /dev/sdXN /mnt
   sudo mount --bind /dev /mnt/dev
   sudo mount --bind /proc /mnt/proc
   sudo mount --bind /sys /mnt/sys
   sudo chroot /mnt
   ```
4. 重置密码：
   ```bash
   passwd <用户名>
   ```
5. 退出并清理挂载：
   ```bash
   exit
   sudo umount /mnt/sys; sudo umount /mnt/proc; sudo umount /mnt/dev; sudo umount /mnt
   ```

> ⚠️ 若目标系统启用 **LUKS 全盘加密**且无恢复密钥，chroot 能改密码但**无法解密数据盘**——数据依旧加密，需密钥。[digibeatrix](https://www.digibeatrix.com/ubuntu/en/troubleshooting-en/recover-ubuntu-password-en/)

---

# 场景 B：离线破解 hash 哈希（授权 / CTF / 实验室）

适用：你拿到了一份明文可读的 `/etc/shadow`（或其中单个哈希），想还原其明文口令。

## B1. 准备工具与字典

安装 John the Ripper（**jumbo 版才支持现代哈希**）：
```bash
sudo apt update
sudo apt install -y john
```
安装 hashcat（GPU 破解）：
```bash
sudo apt install -y hashcat
# 或从官网拉二进制：https://hashcat.net/hashcat/
```
字典 rockyou（约 134MB、1400 万行）：
```bash
# Ubuntu/Debian 环境下可由下列任一途径获取
curl -L -o /tmp/rockyou.txt https://github.com/zacheller/rockyou/raw/master/rockyou.txt
# Kali 系默认在：
#   /usr/share/wordlists/rockyou.txt
```

## B2. 把 shadow 转成工具可用格式

```bash
# 备份并合并 /etc/passwd + /etc/shadow
sudo unshadow /etc/passwd /etc/shadow > hashes.txt
```
- **John** 直接吃 `hashes.txt`。
- **hashcat** 通常需要**只保留哈希那一行**的 `用户名:$hash$...` 形式（去掉 `:` 后的其它字段）。

## B3. 用 John the Ripper 破解

```bash
# 自动识别哈希
john --wordlist=/tmp/rockyou.txt --rules=Wordlist hashes.txt

# 若自动识别失败，显式指定 crypt 家族
john --format=crypt --wordlist=/tmp/rockyou.txt hashes.txt
```
快速查看已破出的口令：
```bash
john --show hashes.txt
```

> ⚠️ **yescrypt（`$y$`）**：仅 **jumbo** 构建支持，且个别发行版 jumbo 存在未编译支持的问题。若提示 “No password hashes loaded”，请确认：①你用的是 jumbo 而非 core john；②本次构建确实包含 yescrypt。[EXAMPLES](https://github.com/openwall/john/blob/bleeding-jumbo/doc/EXAMPLES)

## B4. 用 hashcat 破解（GPU 加速更快）

> mode 数字以本机 `hashcat -h` / 官方 wiki 为准；`$6$`=1800、`$1$`=500 是公认值。**yescrypt（`$y$`）暂无独立 GPU mode**，通常需 CPU 参考实现/桥接，速度慢，教程如实说明，不要臆造 mode 号。[hashcat wiki](https://hashcat.net/wiki/doku.php?id=hashcat)

```bash
# SHA-512-crypt（$6$），GPU
hashcat -m 1800 -a 0 hashes.txt /tmp/rockyou.txt

# MD5-crypt（$1$）
hashcat -m 500  -a 0 hashes.txt /tmp/rockyou.txt

# 仅用 CPU（-d 0 手动指定设备）
hashcat -m 1800 -a 0 hashes.txt /tmp/rockyou.txt -d 0
```

### 提高命中的策略

| 需求 | hashcat 参数 | 说明 |
|---|---|---|
| 纯字典攻击 | `-a 0 ... rockyou.txt` | 最基础 |
| 字典 + 规则 | `-a 0 -r best64.rule hashes.txt rockyou.txt` | 加变形规则突破常见套路 |
| 纯掩码暴破 | `-a 3 hashes.txt ?a?a?a?a?a?a` | 6 位纯符号/大小写数字 |
| 纯数字暴破 | `-a 3 hashes.txt ?d?d?d?d?d?d` | 6 位纯数字 |
| 字典 + 掩码（混合） | `-a 6 hashes.txt /tmp/rockyou.txt ?d?d?d` | 如 `pass123` 这类 |

`?d`=数字、`?l`=小写、`?u`=大写、`?a`=全字符。[hashcat examples](https://kennyvn.com/hashcat-tutorial-kali-linux/)

### John 对应策略

```bash
john --wordlist=/tmp/rockyou.txt --rules=Jumbo hashes.txt   # jumbo 规则集
john --format=sha512crypt --wordlist=/tmp/rockyou.txt hashes.txt
```

## B5. 工具对比与选择

| 维度 | John the Ripper (jumbo) | hashcat |
|---|---|---|
| 易用性 | 高，自动识别哈希 | 中，需手动指定 mode |
| CPU 破解 | 好 | 好（`-d 0`） |
| GPU 加速 | 一般 | **强项** |
| yescrypt `$y$` | jumbo 支持（看构建） | 无独立 GPU mode |
| 输入格式 | `unshadow` 输出可直接用 | 需精简为 `user:hash` |

**建议**：先 John 快速跑常见字典；跑不动再上 hashcat + 显卡 + 规则。

---

## 4. 安全加固建议（防御侧，逆向加固）

既然知道了口令如何被攻破，就应堵死这些口子：

1. **改用强口令**：≥12 位、混合大小写/数字/符号，避免在 rockyou / 常见规则内。
2. **禁用/轮换高频弱口令**：定期 `chage` 强制改密。
3. **开启 LUKS 全盘加密**：即便拿到磁盘也解不开数据。
4. **对 GRUB 设置密码保护**，防止他人进 recovery 改你密码：
   ```bash
   grub-mkpasswd-pbkdf2          # 生成一个 pbkdf2 哈希
   ```
   把输出写入 `/etc/grub.d/40_custom` 的 `set superusers` / `password_pbkdf2` 中，再 `sudo update-grub`。[Veeam](https://www.veeam.com/blog/ubuntu-linux-defense-secure-boot-single-user.html)
5. **加固 BIOS/UEFI**：设开机密码、关闭从 USB 启动，防 Live USB chroot。
6. **权限最小化**：`/etc/shadow` 仅 root 可读（默认 `640 root:shadow`），别放松。

---

## 5. 常见问题（QA）

**Q1：`john` 报 “No password hashes loaded”？**
多半不是 jumbo 版，或 yescrypt 未编译。`john --list=formats | grep -i yescrypt` 确认支持。

**Q2：hashcat 提示 “No hashes loaded”？**
-check 哈希格式，确保只有 `user:$hash`（去掉其它 `:` 字段），并核对 mode 正确。

**Q3：改了密码后原来的加密 /home 还能进吗？**
密码只控制登录；若数据未单独加密（LUKS），重置后原数据仍在，可直接登录访问。若 /home 用 ecryptfs 等按口令加密，则重置密码会**失去解密密钥**，数据不可恢复——务必先备份。

**Q4：`passwd -d` 清空密码安全吗？**
仅临时方便找回，清空后**任何本地人可直接空密码登录**，应尽快重设强口令。

---

## 参考来源

- Ubuntu RecoveryMode：https://wiki.ubuntu.com/RecoveryMode
- Ubuntu LostPassword：https://help.ubuntu.com/community/LostPassword
- Ubuntu password-hashing：https://documentation.ubuntu.com/security/docs/security-forms/cryptography/password-hashing/
- crypt(5) 手册：https://manpages.ubuntu.com/manpages/focal/man5/crypt.5.html
- John the Ripper 官方 EXAMPLES/OPTIONS：https://github.com/openwall/john/blob/bleeding-jumbo/doc/EXAMPLES
- hashcat wiki：https://hashcat.net/wiki/doku.php?id=hashcat
- rockyou 字典（Kali 页）：https://www.kali.org/tools/wordlists/
- Veeam 单用户模式与 GRUB 加固：https://www.veeam.com/blog/ubuntu-linux-defense-secure-boot-single-user.html

---

> 本教程旨在帮助系统管理员找回**本人或已授权**的 Ubuntu 登录口令。请勿用于任何未授权系统。