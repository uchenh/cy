# Ubuntu：`sudo reboot` 无效（只列出 systemctl 单元）排查与修复

> 关联问题：同一台机器曾感染 CPU 挖矿木马，并篡改系统命令  
> 现象：`sudo reboot` 不重启，而是刷出整屏 `systemctl` 的 UNIT 列表  
> 结论：`/bin/systemctl` 被替换为静态链接假文件，并加了 `+i`/`+a` 保护；`reboot` 的软链接路径本身正常

---

## 1. 现象

执行：

```bash
sudo reboot
```

预期：系统重启。  

实际：输出大量 `systemctl list-units` 风格内容（`UNIT` / `LOAD` / `ACTIVE` / `SUB`），**机器不重启**。

可用变通方式：

```bash
sudo systemctl reboot    # 此命令在修复前通常仍可用
sudo shutdown -r now
```

---

## 2. 原因说明

Ubuntu 上正常布局为：

```text
/sbin/reboot      -> /bin/systemctl
/usr/sbin/reboot  -> /bin/systemctl
```

官方 `systemctl` 是 **多调用二进制**：当 `argv[0]` 为 `reboot` / `poweroff` / `halt` 时，会执行对应电源操作。

本机异常点不在软链接，而在 **`/bin/systemctl` 本体被木马替换**：

| 对比项 | 假文件（被篡改） | 官方包内文件 |
|--------|------------------|--------------|
| 链接方式 | **statically linked** | **dynamically linked**（PIE） |
| 体积（示例） | 约 1311768 字节 | 约 1115760 字节 |
| 文件属性 | `----ia--------e-------`（含 **i** 不可变、**a** 只追加） | 正常，无 i/a |
| 行为 | 经 `reboot` 调用时不重启，等同裸跑 `systemctl` 列服务 | `reboot` / `systemctl reboot` 均可重启 |
| BuildID | `036c4ac8…`（示例） | `4e7e6e73…`（示例，随包版本变化） |

因此：

- `sudo reboot` → 走假 `systemctl`，且不按名字执行 reboot → 只列单元  
- `sudo systemctl reboot` → 显式子命令，假程序仍可能实现了该路径 → 能重启  

此前 `apt`/`dpkg` 升级 `systemd` 失败（`unable to make backup link of './bin/systemctl' ... Operation not permitted`），也是因为假文件带 **`+i`/`+a`**，无法被覆盖。

---

## 3. 排查命令

```bash
type reboot
ls -la /sbin/reboot /usr/sbin/reboot
file /sbin/reboot /bin/systemctl
lsattr /bin/systemctl

# 与官方包对比
cd /tmp
apt-get download systemd
dpkg-deb -x systemd_*.deb /tmp/systemd-extract
file /tmp/systemd-extract/bin/systemctl /bin/systemctl
ls -la /tmp/systemd-extract/bin/systemctl /bin/systemctl
```

判断要点：

1. 软链接指向 `/bin/systemctl` → **正常**，不要误改链接  
2. `/bin/systemctl` 为 **static**，包内为 **dynamic** → 已被替换  
3. `lsattr` 出现 **`i`** 或 **`a`** → 需先去保护再替换  

---

## 4. 修复步骤（已验证可用）

> 需已具备可用的真 `chattr`（若 `chattr` 也曾被篡改，需先从 `e2fsprogs`  deb 解压恢复）。

```bash
# 1) 去掉假 systemctl 的 i/a 等保护
sudo chattr -iauc /bin/systemctl
sudo lsattr /bin/systemctl
# 期望类似：--------------e-------

# 2) 下载并解压官方 systemd（若尚未解压）
cd /tmp
apt-get download systemd
dpkg-deb -x systemd_*.deb /tmp/systemd-extract
file /tmp/systemd-extract/bin/systemctl
# 确认：dynamically linked

# 3) 备份假文件并替换
sudo cp -a /bin/systemctl /bin/systemctl.fake.bak
sudo cp -a /tmp/systemd-extract/bin/systemctl /bin/systemctl
sudo chown root:root /bin/systemctl
sudo chmod 755 /bin/systemctl

# 4) 验证
file /bin/systemctl
lsattr /bin/systemctl
ls -la /bin/systemctl
/usr/sbin/reboot --help 2>&1 | head -5
```

修复成功时，`reboot --help` 应类似：

```text
reboot [OPTIONS...] [ARG]

Reboot the system.
...
```

而不再刷整屏 UNIT 列表。

可选：继续用包管理对齐整个 `systemd`：

```bash
sudo apt --fix-broken install
sudo apt-get install --reinstall systemd
```

确认无问题后，可删除备份：

```bash
sudo rm -f /bin/systemctl.fake.bak
```

---

## 5. 修复后用法

以下应均可正常工作：

```bash
sudo reboot
sudo systemctl reboot
sudo shutdown -r now
sudo systemctl poweroff
```

---

## 6. 小结

| 项目 | 内容 |
|------|------|
| 直接原因 | `/bin/systemctl` 被替换为静态链接假二进制 |
| 保护手段 | ext 属性 `+i`、`+a`，阻止 dpkg/删除 |
| 软链接 | `/sbin/reboot → /bin/systemctl` **无需修改** |
| 临时办法 | `sudo systemctl reboot` |
| 彻底修复 | `chattr -iauc` 后用官方包内 `systemctl` 覆盖，并 `chown root:root` |

该类篡改常与挖矿/后门共存；若还发现假 `chattr`、异常 `systemd` 服务等，应一并清理，并评估是否需要重装系统。

---

*文档仅记录本机一次实际修复过程，包版本与 BuildID 以当时 `jammy-updates` 的 `systemd 249.11-0ubuntu3.21` 为例，其它环境请以 `apt-get download` 解压结果为准。*
