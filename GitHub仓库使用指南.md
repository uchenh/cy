# GitHub 仓库 cy 克隆与日常使用指南

> 仓库地址：`https://github.com/uchenh/cy`
> 本指南说明如何在 Windows（Git Bash 环境）下把该仓库克隆到本地，并进行日常的"更新、上传、修改、删除"等操作。
> 目标目录：`C:\Users\yuchen\Desktop\工作目录\github`

---

## 0. 现状说明（已替你核实）

| 项目 | 状态 |
|---|---|
| 远程仓库 | ✅ 公开存在，默认分支 `main`，内含 教程/模型部署/代理/其他/知识 等目录 |
| 本地 Git | ✅ 已安装 Git 2.55（Git Bash） |
| 全局身份 | ✅ 已配置 `user.name=chenyu686`，`user.email=15374324849@163.com` |
| Git 代理 | ✅ 已配置走本机 Clash（`http://127.0.0.1:7897`），**克隆前无需再手动配置** |
| 目标目录 | ✅ `C:\Users\yuchen\Desktop\工作目录\github` 已存在（当前为空） |

> ⚠️ 提示：全局配置的邮箱用于**提交记录署名**；推送到 GitHub 时登录验证用的是你的 **GitHub 账号**，两者不是一回事。

---

## 1. 首次克隆仓库

### 1.1 打开 Git Bash

开始菜单搜索"Git Bash"打开，或在目标目录里右键 → **Open Git Bash here**。

### 1.2 克隆前：确认 Git 已走代理（国内网络必读）

> 先回答你的问题：**不需要先在终端窗口里单独"配代理"再克隆**。Git 使用代理的顺序是：**Git 自身配置（全局，已替你配好）→ 当前窗口临时环境变量 → 无代理**。三种方式的区别如下：

| 方式 | 在哪里配置 | 作用范围 | 你需要做吗 |
|---|---|---|---|
| ① Git 全局代理（推荐） | `~/.gitconfig`（`git config --global ...`） | 所有 Git 操作、所有窗口、**永久生效** | ✅ **已替你配好**，保持 Clash 开启、直接克隆即可 |
| ② 当前窗口临时代理 | 在 Git Bash 窗口里 `export ...` | **只对当前这一个窗口生效**，关掉就失效 | 不用；仅当想临时换代理/覆盖时用 |
| ③ Windows 系统代理 | 设置 → 网络和 Internet → 代理 | 只影响浏览器等软件，**Git 不走它** | 不用管 |

如果你更习惯"窗口里先配代理再克隆"的方式（⚠️ 关窗后失效，下次需重新执行）：

```bash
# 在 Git Bash 窗口里依次执行
export http_proxy=http://127.0.0.1:7897
export https_proxy=http://127.0.0.1:7897
git clone https://github.com/uchenh/cy.git
```

克隆前可先验证代理配置是否生效（方式①已配好时应输出 `http://127.0.0.1:7897`）：

```bash
git config --global --get https.proxy
```

> ⚠️ 前提：**保持 Clash 类代理软件开启**。若报"端口连接失败/拒绝"，说明代理软件没开或换了端口，把命令里的 `7897` 改成你实际的代理端口即可。

### 1.3 执行克隆

```bash
cd /c/Users/yuchen/Desktop/工作目录/github   # 进入目标目录
git clone https://github.com/uchenh/cy.git    # 克隆仓库
cd cy                                         # 进入克隆下来的仓库目录
```

执行后会在 `github` 目录下生成 `cy` 文件夹，里面就是完整的仓库内容：

```
C:\Users\yuchen\Desktop\工作目录\github\
└── cy\
    ├── README.md
    ├── 代理\
    ├── 其他\
    ├── 教程\
    │   ├── ceph\
    │   ├── lustre\
    │   └── slurm集群部署\
    ├── 模型部署\
    └── 知识\
```

### 1.4 第一次推送需要登录（任选一种，推荐方式一）

GitHub 已于 2021 年起**取消密码推送**，必须用 Token 或 SSH 或 GitHub CLI：

**方式一：Personal Access Token（PAT，最常用）**

✅ **本机已完成浏览器授权**（Git Credential Manager），日常 push / pull 全部免密，**不需要再使用令牌**。

如果以后出现 `Authentication failed`、或换机器需要令牌，按下面步骤重新生成（⚠️ **生成后不要写进任何文档/仓库**）：

1. 浏览器打开 <https://github.com/settings/tokens> → Generate new token (classic)
2. 勾选 `repo` 权限，生成后**立即复制保存**（只显示一次）
3. 推送时提示输入用户名 → 输入 `uchenh`；提示输入密码 → **粘贴刚生成的令牌**（粘贴后界面不显示字符是正常的）→ 回车
4. 成功后凭据会被 Git Credential Manager 记住，之后 push / pull 免密。

**方式二：GitHub CLI（一劳永逸）**

```bash
gh auth login    # 按提示选 GitHub.com → HTTPS → 浏览器登录即可
```

登录一次后，push/pull 全程免密。

---

## 2. 日常使用流程

### 2.1 每天开工前：先更新到最新

```bash
cd /c/Users/yuchen/Desktop/工作目录/github/cy
git pull
```

> `git pull` = 把远程最新提交拉下来并自动合并。如果本地没改过东西，永远安全；如果本地有未提交的修改且与远端冲突，会报错，见第 4 节。

### 2.2 上传"新文件"到仓库

三步曲：**add → commit → push**。

```bash
git status              # ① 先看有哪些改动（红色 = 未提交的新文件/修改）
git add .               # ② 把当前目录所有改动加入暂存区（也可 git add 具体文件路径）
git commit -m "docs: 新增xxx安装教程"   # ③ 提交，写清这次干了什么
git push                # ④ 推送到 GitHub
```

> 建议：把要上传的文档放进对应分类目录（如部署类放 `教程/`），再执行上面的命令。

### 2.3 修改已有文件后上传

和上面完全一样，改完文件后执行 `git status` → `git add` → `git commit -m "update: ..."` → `git push`。

### 2.4 删除 / 重命名文件

```bash
git rm 教程/旧文件.md            # 删除文件
git mv 教程/旧文件.md 教程/新文件.md   # 重命名
git commit -m "chore: 删除过时文档"
git push
```

### 2.5 常用查看命令

```bash
git status                 # 当前工作区状态（最常用）
git log --oneline -10      # 最近 10 条提交历史
git log --oneline -5 --stat # 最近提交改了哪些文件
git diff                   # 看未提交的具体改动内容
```

---

## 3. 关于 .gitignore（重要）

- 仓库根目录下放一个 `.gitignore` 文件，可让某些文件**永不参与上传**。例如：

```gitignore
# 说明类文件（不要上传）
*使用指南.md
# 临时文件
~$*.docx
Thumbs.db
```

- 注意：**`git add .` 会把目录里所有没被忽略的文件都加进去**。建议把本指南这类"说明文件"放在仓库目录**外面**（本文件就放在 `github` 文件夹的同级，不在 `cy` 仓库内），避免误传。
- GitHub 单文件上限 100MB，超过会 push 失败，大文件要用 Git LFS 或拆分。

---

## 4. 常见问题排查

| 现象 | 原因 | 解决办法 |
|---|---|---|
| `Authentication failed` / `could not read Username` | 未配置推送凭证 | 按 1.4 方式一/方式二配置一次 |
| `Please tell me who you are` | 没配提交身份 | `git config --global user.name "chenyu686"` + `git config --global user.email "你的邮箱"` |
| `failed to push some refs` / `non-fast-forward` | 远端有新提交，本地落后 | 先 `git pull`，再重新 `git push` |
| 推送被拒（pull 后仍冲突） | 本地与远端改了同一处 | 按下方 4.1 处理冲突 |
| 中文文件名显示成 `\345\255\246` | Git 默认转义中文 | `git config --global core.quotepath false` |
| `LF will be replaced by CRLF` 警告 | Windows 换行符差异 | 无害警告，可忽略；介意可执行 `git config --global core.autocrlf true` |
| 误提交了不该传的文件 | 想撤回 | `git rm --cached 文件名` 后 commit，保留本地文件只移除跟踪 |

### 4.1 冲突处理（最简流程）

```bash
git pull                    # 冲突时 Git 会提示哪些文件冲突
# 用编辑器打开冲突文件，保留需要的内容，删掉 <<<<<<< ======= >>>>>>> 标记行
git add 冲突文件            # 标记"已解决"
git commit -m "merge: 解决冲突"
git push
```

> 多数情况下冲突只发生在多人同时改同一文件。个人仓库若先 `git pull` 再改再传，基本不会遇到。

### 4.2 克隆失败 / 卡住（国内网络访问 GitHub 问题）

症状：`git clone` 长时间无响应、`Failed to connect`、`fatal: unable to access ...`、`RPC failed` 等。

**根本原因**：GitHub 在国内直连不稳定/被墙。解决办法是让 Git 走你的代理软件（Clash/v2ray 等）。

✅ **本机已配置完成**：已检测到你本机 7897 端口有 Clash 类代理在运行，并已写入 Git 全局配置：

```ini
[http]
	proxy = http://127.0.0.1:7897
[https]
	proxy = http://127.0.0.1:7897
```

现在只需**保持代理软件开启**，直接重新克隆即可：

```bash
cd /c/Users/yuchen/Desktop/工作目录/github
git clone https://github.com/uchenh/cy.git
```

如需手动配置/验证/撤销：

```bash
# 验证是否生效
git config --global --get https.proxy    # 应输出 http://127.0.0.1:7897

# 手动配置
git config --global http.proxy http://127.0.0.1:7897
git config --global https.proxy http://127.0.0.1:7897

# 撤销代理（换端口或不想走代理时）
git config --global --unset http.proxy
git config --global --unset https.proxy
```

> 注意：该全局代理也会作用于 Gitee 等其它仓库，通常无影响（代理软件一般自动直连国内站点）；若发现 Gitee 推送变慢，可按上面"撤销代理"命令移除，仅对 GitHub 单独配置（去掉 `--global`，在仓库目录内执行即为单仓库配置）。

**备用方案（不开代理也能拉）**：

1. **镜像站克隆**（任选一个试，镜像可用性会变动）：
   ```bash
   git clone https://gitclone.com/github.com/uchenh/cy.git
   # 或
   git clone https://kkgithub.com/uchenh/cy.git
   ```
2. **浏览器下载 ZIP**：在能打开 GitHub 网页的情况下，访问
   `https://github.com/uchenh/cy/archive/refs/heads/main.zip`
   下载后解压，把解压出来的 `cy-main` 文件夹改名为 `cy` 即可（这种方式的缺点是以后无法 `git pull` 增量更新，需重新下载，仅应急用）。

---

## 5. 命令速查表

| 场景 | 命令 |
|---|---|
| 克隆仓库 | `git clone https://github.com/uchenh/cy.git` |
| 进入仓库 | `cd /c/Users/yuchen/Desktop/工作目录/github/cy` |
| 查看状态 | `git status` |
| 更新到最新 | `git pull` |
| 查看改动内容 | `git diff` |
| 加入暂存（全部） | `git add .` |
| 加入暂存（单个） | `git add 教程/xx.md` |
| 提交 | `git commit -m "说明文字"` |
| 推送 | `git push` |
| 查看历史 | `git log --oneline -10` |
| 删除文件 | `git rm 路径` → commit → push |
| 重命名 | `git mv 旧路径 新路径` → commit → push |

---

## 6. 推荐日常节奏（打卡模板）

```bash
# 开工
cd /c/Users/yuchen/Desktop/工作目录/github/cy
git pull

# ...写文档 / 把文件放进对应分类目录...

# 收工上传
git status          # 确认要传的内容
git add .
git commit -m "docs: 今天新增了 xxx"
git push
```

记住：**先 pull 再开工，收工前 add + commit + push**，即可保证本地与 GitHub 始终同步。

---

## 附：GitHub 访问令牌（PAT）处置记录——🔴 敏感信息

> **处置说明**：曾有一个 PAT 因误提交进本仓库被 GitHub Push Protection（密钥扫描）拦截，**未上传成功**。为稳妥起见，该令牌已从本文档移除，并**建议立即在 GitHub 上吊销**（视为已泄露处理）。

**当前登录状态：**
- 日常克隆 / 推送：✅ 已走浏览器授权（Git Credential Manager 记住凭据），**全程无需令牌**。
- 若确需新令牌（换机器、凭据丢失）：按上文 **§1.4 方式一** 的步骤重新生成即可。

**吊销旧令牌**：GitHub → 右上角头像 → Settings → Developer settings → Personal access tokens → 找到对应令牌 → Delete。

> 🔴 **安全铁律**：GitHub 令牌等同密码，**永远不要写进任何文档、代码或仓库**（包括本指南）。GitHub 密钥扫描会自动拦截并吊销泄露的令牌；若确需留档，请存在本地密码管理器或仓库外的私密文件中。
