# GitHub 仓库 cy 使用速查（简单版）

- 仓库：`https://github.com/uchenh/cy`　默认分支：**main**
- 本机路径：`C:\Users\yuchen\Desktop\工作目录\github\cy`
- 已配置好：Git、代理（Clash 7897）、登录（浏览器授权，push/pull 免密）

---

## 1. 克隆（仅首次需要）

```bash
cd /c/Users/yuchen/Desktop/工作目录/github
git clone https://github.com/uchenh/cy.git
cd cy
```

> 前提：代理软件（Clash）保持开启。

---

## 2. 提交到哪个分支？

- **默认提交到 `main`**：克隆后本地就在 main 上，`git push` 就是推到远程 main，日常文档直接这么用。
- 查当前分支：`git branch --show-current`
- 如果想另开分支提交（如试验内容）：

```bash
git branch -a
git checkout -b 新分支名        # 新建并切换到新分支
git push -u origin 新分支名      # 首次推送并关联（之后只需 git push）
git checkout main               # 切回 main
```

---

## 3. 日常上传（核心三步）

```bash
cd /c/Users/yuchen/Desktop/工作目录/github/cy
git pull              # 开工前：更新到最新
git add .             # ① 添加改动
git commit -m "说明"   # ② 提交（写明改了什么）
git push              # ③ 推送到当前分支
```

- 先 `git pull` 再开工，收工前 add + commit + push，就不会冲突。
- 删除文件：`git rm 文件路径` → commit → push
- 重命名文件：`git mv 旧路径 新路径` → commit → push

---

## 4. 常用命令速查

| 目的 | 命令 |
|---|---|
| 查看状态/改动 | `git status` / `git diff` |
| 查看历史 | `git log --oneline` |
| 只看某个文件 | `git add 文件路径` |
| 更新 | `git pull` |
| 上传 | `git add .` → `git commit -m "说明"` → `git push` |

---

## 5. 常见问题

| 问题 | 处理 |
|---|---|
| push 被拒（本地落后） | 先 `git pull` 再 `git push` |
| 提示 `Authentication failed` | 浏览器会弹出授权，完成一次即可；或重新生成令牌（不要写进文档/仓库） |
| 中文文件名显示成乱码 | `git config --global core.quotepath false` |
| `LF will be replaced by CRLF` 警告 | 无害，忽略即可 |
| 克隆卡住/失败 | 确认 Clash 开着；或换镜像 `git clone https://gitclone.com/github.com/uchenh/cy.git` |
