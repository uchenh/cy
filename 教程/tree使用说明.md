# tree 命令使用说明

`tree` 用于以**树状结构**递归列出目录下的文件和子目录，方便一眼看清一个目录层级。

***

## 1. 安装

```bash
# Debian / Ubuntu
sudo apt install -y tree

# CentOS / RHEL / Rocky
sudo yum install -y tree

# macOS
brew install tree
```

***

## 2. 基本用法

```bash
tree               # 列出当前目录的树状结构
tree /路径/目录     # 列出指定目录
tree -L 2          # 只显示到第 2 层
tree -d            # 只显示目录，不显示文件
tree -a            # 包含隐藏文件（以 . 开头）
tree -f            # 显示完整路径
tree -h            # 以人类可读方式显示大小（K/M/G）
tree -i            # 不显示缩进线
```

***

## 3. 常用组合（推荐）

| 目的          | 命令                       | <br />  |
| ----------- | ------------------------ | :------ |
| 看目录层级（不含文件） | `tree -d`                | <br />  |
| 只看到 2 层     | `tree -L 2`              | <br />  |
| 含隐藏文件       | `tree -a`                | <br />  |
| 带文件大小       | `tree -h`                | <br />  |
| 显示完整路径      | `tree -f`                | <br />  |
| 排除某些目录      | \`tree -I 'node\_modules | .git'\` |
| 递归全层级       | `tree -L 5`              | <br />  |

> 常用组合示例：`tree -L 3 -d` 表示「深度 3 层的目录结构」，适合快速理解项目骨架。

***

## 4. 排除文件/目录

```bash
# -I 配合正则排除，多个用 | 分隔
tree -I '.git|node_modules|target|__pycache__'

# 直接作用于某个目录
tree /var/lib/docker -L 2
```

***

## 5. 输出到文件

```bash
tree > 目录结构.txt        # 保存当前目录树到文本文件
tree -d > 目录结构.txt     # 只保存目录层级
```

***

## 6. 结合其他命令

```bash
# 统计目录总数
tree -d | grep -c '^├\|^└'    # 粗略统计

# 查看某项目只含源码和配置的层级
find . -maxdepth 2 -not -path './.git*' | sort
```

> 注：`find` 是另一种更灵活的查看方式，但它不安装、默认自带，可替代 `tree` 做简单查看。

***

## 7. 常见问题

| 问题                           | 处理                        |
| ---------------------------- | ------------------------- |
| 提示 `command not found: tree` | 按上面第 1 节安装                |
| 英文目录名显示乱码                    | 正常，与终端编码有关                |
| 只想看目录                        | 用 `tree -d`               |
| 输出太多刷屏                       | 加 `-L 层数` 限制深度，或加 `-I` 排除 |

