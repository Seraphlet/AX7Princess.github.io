---
description: ""
title: "Git学习笔记"
draft: false
date: "2026-07-03T07:50:31+08:00"
slug: "git note"
categories:
 - null
tags:
 - Note
image: ""
---

# Git 命令完全指南：从入门到日常够用的所有指令

## 目录

- [一、配置与初始化](#一配置与初始化)
- [二、日常提交四步曲](#二日常提交四步曲)
- [三、查看状态与差异](#三查看状态与差异)
- [四、分支管理](#四分支管理)
- [五、远程协作](#五远程协作)
- [六、暂存与恢复（stash）](#六暂存与恢复stash)
- [七、撤销与回退](#七撤销与回退)
- [八、标签](#八标签)
- [九、实用技巧](#九实用技巧)
- [十、常见工作流示例](#十常见工作流示例)
- [小结](#小结)

---

## 一、配置与初始化

### `git config` —— 配置用户信息
```bash
git config --global user.name "你的名字"
git config --global user.email "you@example.com"
```
`--global` 对当前用户所有仓库生效；去掉该参数则只对当前仓库生效。第一次提交前必须配置，否则会报错。

### `git init` —— 初始化仓库
```bash
git init
```
在当前目录创建 `.git` 子目录，把它变成一个 Git 仓库。

### `git clone` —— 克隆远程仓库
```bash
git clone https://github.com/user/repo.git
git clone https://github.com/user/repo.git my-folder   # 克隆到指定目录
```
把远程仓库完整下载到本地，并自动建立名为 `origin` 的远程关联。

---

## 二、日常提交四步曲

Git 的核心工作流是：**修改文件 → 暂存 → 提交 → 推送**（可选）。

### `git status` —— 查看当前状态
```bash
git status
```
显示哪些文件被修改、哪些已暂存、哪些未跟踪。最常用的"我当前在哪"命令。

### `git add` —— 暂存改动
```bash
git add file.py          # 暂存单个文件
git add .                # 暂存当前目录所有改动（新增/修改/删除，受 .gitignore 约束）
git add -A               # 暂存整个工作树所有改动
git add -p               # 交互式暂存，按代码块选择
```
`add` 只是把改动放进"暂存区"，还没有真正提交。

### `git commit` —— 提交
```bash
git commit -m "feat: 新增登录功能"
git commit -am "fix: 修复空指针"   # 自动暂存已跟踪文件的修改并提交
```
把暂存区的内容生成一个永久快照（commit）。

### `git push` —— 推送到远程
```bash
git push origin main
git push -u origin main   # 首次推送并关联上游，之后直接 git push
```
把本地提交上传到远程仓库。`-u` 会记下"上游分支"，以后 `git push` / `git pull` 无需再写参数。

---

## 三、查看状态与差异

### `git diff` —— 查看差异
```bash
git diff              # 工作区 vs 暂存区
git diff --staged     # 暂存区 vs 最近一次提交
git diff main backup  # 两个分支之间的差异
```

### `git log` —— 查看提交历史
```bash
git log
git log --oneline        # 一行一条，简洁
git log --oneline -5     # 最近 5 条
git log --graph --all    # 图形化展示分支关系
```

### `git show` —— 查看某次提交详情
```bash
git show <commit-id>
```

### `git blame` —— 逐行追溯
```bash
git blame file.py
```
显示每行代码最后是谁、在哪个提交改的，排查"这段代码谁写的"神器。

---

## 四、分支管理

### `git branch` —— 分支操作
```bash
git branch              # 列出本地分支
git branch feature      # 新建分支
git branch -d feature   # 删除已合并的分支
git branch -D feature   # 强制删除（未合并也删）
```

### `git switch` / `git checkout` —— 切换分支
```bash
git switch main              # 切到 main
git switch -c feature        # 新建并切换到 feature
git checkout main            # 老写法，等价 switch
```
`switch` 是 Git 2.23+ 推荐的新命令，语义比 `checkout`（身兼数职）更清晰。

### `git merge` —— 合并分支
```bash
git switch main
git merge feature     # 把 feature 合并进当前分支
```
可能产生冲突，需手动解决后再 `git add` + `git commit`。

### `git rebase` —— 变基
```bash
git switch feature
git rebase main      # 把 feature 的提交"挪到" main 最新提交之后
```
让提交历史变成一条直线，比 merge 更整洁。但**不要对已经推送到远程的提交做 rebase**，否则会���写共享历史，给协作者制造麻烦。

---

## 五、远程协作

### `git remote` —— 管理远程仓库
```bash
git remote -v                         # 查看已关联的远程
git remote add backup <url>          # 添加第二个远程
git remote remove backup             # 删除远程
```
一个本地仓库可以连接多个远程（例如 `origin` 指向主仓库，`backup` 指向你的备份仓库）。

### `git fetch` —— 拉取但不合并
```bash
git fetch origin
```
把远程更新下载到本地，但不动你的工作区。先 fetch 再查看，比直接 pull 更安全。

### `git pull` —— 拉取并合并
```bash
git pull origin main
git pull --rebase origin main   # 用 rebase 方式拉取，历史更干净
```
等于 `git fetch` + `git merge`。

---

## 六、暂存与恢复（stash）

### `git stash` —— 临时存放改动
```bash
git stash              # 把未提交的改动藏起来
git stash pop          # 恢复最近一次储藏并删除
git stash list         # 查看所有储藏
git stash apply        # 恢复但不删除
```
适合"活干到一半，却要切分支去修紧急 bug"的场景。

---

## 七、撤销与回退

### `git restore` —— 丢弃工作区改动（Git 2.23+）
```bash
git restore file.py          # 撤销单个文件的工作区修改
git restore --staged file.py # 把文件从暂存区移出（取消 add）
```

### `git reset` —— 重置
```bash
git reset --soft HEAD~1   # 回退到上一次提交，改动留在暂存区
git reset --mixed HEAD~1  # 回退，改动留在工作区（默认行为）
git reset --hard HEAD~1   # 彻底回退，丢弃所有改动（危险！）
```
`--hard` 会直接删除未提交的代码，**慎用**。

### `git revert` —— 安全撤销（推荐用于已推送的提交）
```bash
git revert <commit-id>
```
生成一个"反向"的新提交来抵消某次提交，不改写历史，非常适合团队协作场景。

---

## 八、标签

### `git tag` —— 打标签
```bash
git tag v1.0                          # 轻量标签
git tag -a v1.0 -m "发布版本 1.0"     # 附注标签
git push origin v1.0                  # 推送标签到远程
```
常用于标记发布版本。

---

## 九、实用技巧

### `.gitignore` —— 忽略文件
在项目根目录创建 `.gitignore`，写进去的文件不会进入版本控制：
```gitignore
.env            # 密钥/环境变量，禁止提交
node_modules/
__pycache__/
*.log
```
避免把敏感信息（如 `.env` 里的数据库密码）或垃圾文件提交上去。

### 一个 remote 推多个仓库
```bash
git remote set-url --add --push origin https://github.com/你的/AgentLearn.git
```
之后 `git push origin` 会同时推送到配置的所有 push 地址，适合 GitHub + Gitee 双备份。

### 设置别名
```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.lg "log --oneline --graph"
```
之后用 `git st` 代替 `git status`，省时省力。

---

## 十、常见工作流示例

### 日常提交
```bash
git status
git add .
git commit -m "feat: xxx"
git push
```

### Fork 协作
```bash
git clone https://github.com/你的/fork.git
git remote add upstream https://github.com/原作者/repo.git
git fetch upstream
git merge upstream/main
git push origin main
```

### 修 bug 临时切换
```bash
git stash
git switch -c hotfix
# 修复、提交……
git switch main
git merge hotfix
git stash pop
```

## 十一、回退到历史某个版本

代码上线后出问题、或误提交了错误内容时，需要让仓库回到某个旧提交。

### 方式一：reset 回退（本地、未推送时推荐）
```bash
git log --oneline             # 找到目标 commit 的 id
git reset --hard <commit-id>  # 工作区/暂存区/版本库全部回到该版本
git reset --hard HEAD~3       # 回退最近 3 个提交
```
> ⚠️ `--hard` 会丢弃目标版本之后的所有改动，仅在未推送且确定不要时用。

### 方式二：revert 回退（已推送到远程时推荐）
```bash
git revert <commit-id>          # 生成一个反向提交，抵消该次改动
git revert HEAD~2..HEAD         # 批量撤销最近 2 个提交
```
不改写历史，对团队协作安全，之后 `git push` 即可。

### 方式三：临时查看旧版本（不改动当前状态）
```bash
git checkout <commit-id>        # 进入 detached HEAD 查看旧代码
git switch -                    # 看完切回原分支
```

### 回退后强制同步远程
本地 `reset --hard` 回退后若已推送过，需强推（会覆盖远程历史，谨慎）：
```bash
git push --force-with-lease origin main
```
优先用 `--force-with-lease`，比 `--force` 安全：远程有他人新提交时会拒绝，避免误覆盖别人的工作。

### 误删提交找回
```bash
git reflog                     # 查看 HEAD 移动记录，定位丢失的 commit id
git reset --hard <commit-id>   # 恢复
```

## 十二、远程仓库连接与认证管理

### 修改远程仓库地址（改名 / 换平台 / 换账号）
```bash
git remote -v                                          # 查看当前关联
git remote set-url origin https://github.com/新用户名/新仓库名.git
git remote -v                                          # 确认已更新
```
GitHub 改了仓库名或用户名后 URL 随之变化，用 `set-url` 重新指向即可，**本地提交历史全部保留，无需重新 clone**。

### 远程仓库“名字变了”重新连接
仓库名变了本质是远程 URL 变了，处理方法同上：
```bash
git remote set-url origin <新地址>
```
想顺便改本地 remote 的简称（如 `origin` → `github`）：
```bash
git remote rename origin github
```

### HTTPS 与 SSH 互转
```bash
# HTTPS → SSH
git remote set-url origin git@github.com:用户名/仓库名.git

# SSH → HTTPS
git remote set-url origin https://github.com/用户名/仓库名.git
```
- **HTTPS**：每次 push 用个人访问令牌（PAT）或系统凭据管理器认证，最省心
- **SSH**：用本机 SSH 密钥认证，配好后无需每次输密码，适合自动化/脚本

### 生成并配置 SSH 密钥
```bash
ssh-keygen -t ed25519 -C "you@example.com"   # 生成密钥对，一路回车
cat ~/.ssh/id_ed25519.pub                     # 复制公钥内容
# 到 GitHub → Settings → SSH and GPG keys 粘贴添加
ssh -T git@github.com                         # 测试连接，看到成功提示即可
```

### 更换 / 更新 HTTPS 凭据（令牌）
- **Windows**：控制面板 → 凭据管理器 → Windows 凭据 → 找到 `git:https://github.com` → 编辑或删除，下次 push 重新输入
- **命令行清除后重输**：
```bash
git credential-manager reject https://github.com
```

### 验证当前连接方式
```bash
git remote -v                  # URL 是 https:// 还是 git@ 一目了然
ssh -T git@github.com         # 单独验证 SSH 是否连通
```

## 十三、查看与修改 .gitignore 文件

`.gitignore` 决定哪些文件不进版本库。下面是如何查看它、修改它，以及处理"加了规则却没生效"的常见坑。

### 查看 .gitignore
```bash
cat .gitignore                   # 直接查看文件内容
code .gitignore                  # 用编辑器打开（VS Code）
git status --ignored             # 查看当前被忽略的文件列表
```

### 检查某个文件是否被忽略、被哪条规则命中
```bash
git check-ignore -v node_modules   # 显示匹配的规则及其所在行
git check-ignore .env              # 有输出 = 被忽略；无输出 = 未被忽略
```
`-v`（verbose）会告诉你具体是哪一行规则、来自哪个 `.gitignore` 文件，调试"为什么这个文件没被忽略"时极有用。

### 修改 .gitignore
```bash
# 方式一：直接编辑文件（记事本 / VS Code / vim）
# 方式二：命令行追加一条规则
echo "*.log" >> .gitignore
echo ".env"  >> .gitignore
```
常见规则写法：
```gitignore
.env                  # 忽略名为 .env 的文件
.env.*                # 忽略 .env.local 等变体
*.log                 # 忽略所有 .log
node_modules/         # 忽略目录
/build                # 忽略根目录下的 build
```

### 关键坑：文件已被跟踪，加了规则也不生效
如果某文件（如 `.env`）之前已经被 commit 进版本库，后来才加进 `.gitignore`，Git 仍会继续跟踪它。需要先从版本库移除跟踪（**本地文件会保留**）：
```bash
git rm --cached .env          # 从版本库/暂存区移除，但保留本地文件
git commit -m "stop tracking .env"
```
之后 `.gitignore` 才会真正生效，该文件不再被提交推送。

### 强制添加被忽略的文件（临时）
```bash
git add -f important.log      # -f 强制添加，即使被 .gitignore 忽略
```

### 全局忽略（对所有仓库生效）
```bash
git config --global core.excludesfile ~/.gitignore_global
# 之后编辑 ~/.gitignore_global 写入规则，对当前用户所有仓库生效
```

---

## 小结

Git 命令看着多，其实日常 80% 的场景只用 `add` / `commit` / `push` / `pull` / `branch` / `merge` 这几个。先把它们用熟，剩下的 `rebase`、`stash`、`revert` 等按需查即可。建议把本文收藏，遇到忘记的命令随时翻。

**记住一条铁律**：还没推送到远程的提交，可以用 `reset --hard` 删；已经推送出去的，请用 `revert` 安全撤销。
