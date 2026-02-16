---
title: git
abbrlink: 69c3279c
date: 2024-10-10 09:13:41
updated:
tags: "git"
categories:
keywords:
description:
top_img: false
cover:
highlight_shrink:
---

> 推荐学习官方文档 `https://git-scm.com/book/zh/v2/`

## **起步**

版本控制是一种记录一个或若干文件内容变化，以便将来查阅特定版本修订情况的系统。

在分布式版本控制系统（DVCS）中，客户端并不只提取最新版本的文件快照，而是把代码仓库完整地镜像下来，包括完整的历史记录。因此，任何一处协同工作用的服务器发生故障，事后都可以用任何一个镜像出来的本地仓库恢复。每一次的克隆操作，实际上都是一次对代码仓库的完整备份。

更进一步，借助这类系统可以指定和若干不同的远端代码仓库进行交互。籍此，在同一个项目中能够分别和不同工作小组的人相互协作。

---

### **安装**

#### **Windows**

推荐使用 `scoop` 来管理 Windows 下的软件，可以直接使用命令安装 `git`：

```powershell
scoop install git
```

#### **Ubuntu**

不建议源代码安装，因为 `git` 会周期性推送安全更新。在 Ubuntu 中直接安装：

```shell
sudo apt install git
```

对于想要体验最新稳定版本，使用此 PPA：

```shell
sudo add-apt-repository ppa:git-core/ppa
sudo apt update && sudo apt install git
```

---

### **初次配置**

初次运行 `git` 前需要进行配置。每台计算机只需要配置一次，程序升级时会保留配置信息。

`git` 自带一个 `git config` 工具来帮助设置控制 Git 外观和行为的配置变量。这些变量存储在三个不同的位置：

1. `/etc/gitconfig`：包含系统上每一个用户及他们仓库的通用配置。使用 `--system` 选项读写该文件（需要管理员权限）。
2. `~/.gitconfig` 或 `~/.config/git/config`：只针对当前用户。使用 `--global` 选项读写此文件，对系统上**所有**仓库生效。
3. 当前仓库的 `.git/config`：仅针对该仓库。使用 `--local` 选项读写此文件（默认行为）。

每一个级别会覆盖上一个级别的配置，所以 `.git/config` 的配置变量会覆盖 `/etc/gitconfig` 中的配置变量。

在 Windows 中，这些文件的位置分别为：

1. `C:\ProgramData\Git\config`（如果使用 scoop 安装，则配置文件目录被修正为 scoop 文件夹）
2. `C:\Users\$User\.gitconfig`
3. 当前仓库中的 `.git/config`

通过以下命令查询所有的配置以及它们所在的文件：

```shell
git config --list --show-origin
```

#### **用户信息**

设置用户名和邮件地址，每一个 `git` 提交都会使用这些信息，它们会写入到每一次提交中，不可更改：

```shell
git config --global user.name "username"
git config --global user.email "user@example.com"
```

使用 `git config --list` 来列出所有 Git 当前能找到的配置。

---

## **Git 基础**

只有在命令行模式下才能执行 Git 的所有命令。

### **获取 Git 仓库**

通常有两种方式：

1. 将尚未进行版本控制的本地目录转换为 Git 仓库；
2. 从其他服务器克隆一个已存在的 Git 仓库。

在已存在目录中初始化仓库：

```shell
git init
```

这将创建 `.git` 目录，包含初始化的 Git 仓库中所有的必须文件。

克隆现有的仓库：

```shell
git clone https://github.com/libgit2/libgit2
```

这会在当前目录下创建名为 `libgit2` 的文件夹，并从远程仓库中拉取所有数据到该目录下的 `.git` 文件夹，然后检出最新版本的文件拷贝。

希望在克隆时自定义本地仓库的名字，可以通过额外参数指定：

```shell
git clone https://github.com/libgit2/libgit2 mylibgit
```

---

### **记录更新**

工作目录下的所有文件都有两种状态：**已跟踪**或**未跟踪**。

- `git status` 用于查看文件处于什么状态。
- `git add <file>` 用于将内容添加到下一次提交中。可以用它开始跟踪新文件、把已跟踪的文件放到暂存区，还能用于合并时把有冲突的文件标记为已解决状态。

### **忽略文件**

总有些文件无需纳入 Git 的管理，也不希望它们总出现在未跟踪文件列表。在这种情况下，可以创建一个名为 `.gitignore` 的文件，列出要忽略的文件的模式。

`.gitignore` 的格式规范：

- 所有空行或以 `#` 开头的行都会被 Git 忽略
- 可以使用标准的 glob 模式匹配，它会递归地应用在整个工作区中
- 匹配模式可以以 `/` 开头防止递归
- 匹配模式可以以 `/` 结尾指定目录
- 要忽略指定模式以外的文件或目录，可以在模式前加上 `!` 取反

一个实用的 `.gitignore` 示例：

```
# 编译产物
*.o
*.class
/build/
/dist/

# 依赖目录
node_modules/
vendor/

# 环境配置
.env
.env.local

# IDE 文件
.idea/
.vscode/
*.swp
```

> GitHub 维护了一份针对各种语言和框架的 `.gitignore` 模板：`https://github.com/github/gitignore`

---

### **查看修改**

- `git diff` 比较工作目录中当前文件和暂存区域快照之间的差异。
- `git diff --staged` 查看已暂存文件与最后一次提交的文件差异。

### **提交更新**

- `git commit` 提交暂存区的内容，会启动文本编辑器来输入提交说明。
- `git commit -m "message"` 直接在命令行中填写提交说明。
- `git commit -a` 跳过暂存区域，将所有已跟踪文件暂存起来一并提交，从而跳过 `git add` 步骤。

### **移除文件**

- `git rm <file>` 将文件移除，在下一次提交时不再纳入版本管理。
- `git rm --cached <file>` 将文件从 Git 仓库中删除，但仍然保留在当前工作目录中。
- `git rm` 命令支持 **glob** 模式。

### **移动文件**

Git 并不显式跟踪文件移动操作。如果在 Git 中重命名了某个文件，仓库的元数据并不会体现这是一次改名操作。

`git mv file_from file_to` 用于在 Git 中对文件改名，这相当于执行了：

```shell
mv file_from file_to
git rm file_from
git add file_to
```

---

### **查看提交历史**

`git log` 用于回顾提交历史。不传入参数的情况下，会按照时间先后顺序列出所有提交，最近的更新排在最上面。

常用选项：

| 选项 | 说明 |
|---|---|
| `--patch` / `-p` | 显示每次提交所引入的差异 |
| `-n` | 只显示最近的 n 次提交 |
| `--stat` | 显示每次提交的简略统计信息 |
| `--oneline` | 将每个提交压缩到一行显示 |
| `--graph` | 以 ASCII 图形展示分支和合并历史 |
| `--pretty=format:"..."` | 自定义输出格式 |
| `--since` / `--until` | 按时间筛选 |
| `--author` | 指定作者 |
| `--grep` | 搜索提交说明中的关键字 |
| `-S <string>` | 筛选添加或删除了该字符串的提交 |

自定义格式示例：

```shell
$ git log --pretty=format:"%h - %an, %ar : %s"
ca82a6d - Scott Chacon, 6 years ago : changed the version number
085bb3b - Scott Chacon, 6 years ago : removed unnecessary test
a11bef0 - Scott Chacon, 6 years ago : first commit
```

> 这里特别提及作者和提交者之间的区别：作者是实际上做出修改的人，提交者是最后将此工作提交到仓库的人。当你为某个项目发布补丁，然后某个核心成员将你的补丁并入项目时，你就是作者，而那个核心成员就是提交者。

常用的组合命令：

```shell
# 简洁的图形化日志
git log --oneline --graph --all

# 查看某个文件的修改历史
git log --follow -p <file>
```

---

### **撤销操作**

在任何一个阶段，你都有可能想要撤销某些操作。**注意，有些撤销操作是不可逆的。**

#### **修改最后一次提交**

提交时发现有几个文件漏掉或提交信息写错了，可以运行带有 `--amend` 选项的提交来重新提交：

```shell
git commit --amend
```

这个命令将暂存区中的文件提交，替换上一次的提交。

#### **取消暂存文件**

```shell
# 传统方式
git reset HEAD <file>

# 推荐的新方式（Git 2.23+）
git restore --staged <file>
```

`reset` 是一个危险的命令，尤其在使用 `--hard` 选项时。

#### **撤销对文件的修改**

```shell
# 传统方式
git checkout -- <file>

# 推荐的新方式（Git 2.23+）
git restore <file>
```

这会使用最近提交的版本来覆盖文件，本地的修改将丢失。

---

### **远程仓库的使用**

远程仓库是指托管在其他网络中的项目的版本库（远程仓库也可以在你的本地主机上）。

常用命令：

| 命令 | 说明 |
|---|---|
| `git remote` | 查看已配置的远程仓库服务器 |
| `git remote -v` | 显示远程仓库的简写与对应的 URL |
| `git remote add <name> <url>` | 添加一个新的远程仓库 |
| `git remote show <name>` | 查看某个远程仓库的详细信息 |
| `git remote rename <old> <new>` | 重命名远程仓库 |
| `git remote remove <name>` | 移除远程仓库 |
| `git fetch <remote>` | 从远程仓库拉取数据（不自动合并） |
| `git pull` | 拉取并自动合并到当前分支 |
| `git push <remote> <branch>` | 将本地分支推送到远程仓库 |

---

### **打标签**

Git 可以给仓库历史中的某一个提交打上标签，以示重要。通常用来标记发布结点（v1.0、v2.0 等）。

```shell
# 列出标签
git tag
git tag -l "v1.8.5*"    # 按模式匹配

# 创建附注标签（推荐，包含完整信息）
git tag -a v1.4 -m "my version 1.4"

# 创建轻量标签
git tag v1.4-lw

# 对过去的提交打标签
git tag -a v1.2 <commit-hash>

# 推送标签到远程
git push origin v1.4       # 推送单个标签
git push origin --tags     # 推送所有标签

# 删除标签
git tag -d v1.4            # 删除本地标签
git push origin --delete v1.4  # 删除远程标签
```

> 使用 `git checkout <tag>` 检出标签会使仓库处于"分离头指针"（detached HEAD）状态。如果需要在此基础上修改，应该创建一个新分支：`git checkout -b <branch> <tag>`。

---

### **Git 别名**

通过 `git config` 命令来为每个命令设置别名：

```shell
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
git config --global alias.lg "log --oneline --graph --all"
```

这意味着，当要使用 `git commit` 时，只需要输入 `git ci`。

---

## **Git 分支**

分支是 Git 最强大的特性之一。Git 的分支模型极其轻量，创建和切换分支几乎是瞬间完成的，这鼓励开发人员频繁地使用分支。

### **分支原理**

Git 的分支本质上是指向提交对象的可变指针。默认分支名为 `main`（或 `master`）。每次提交时，当前分支指针会自动向前移动。

Git 通过一个名为 `HEAD` 的特殊指针来标识当前所在的本地分支。

### **基本操作**

```shell
# 创建分支
git branch <branch-name>

# 切换分支
git checkout <branch-name>

# 创建并切换（推荐，Git 2.23+）
git switch -c <branch-name>

# 查看所有分支
git branch          # 本地分支
git branch -a       # 包括远程分支
git branch -v       # 显示每个分支的最后一次提交

# 删除分支
git branch -d <branch-name>    # 安全删除（已合并才允许）
git branch -D <branch-name>    # 强制删除

# 重命名当前分支
git branch -m <new-name>
```

---

### **合并分支**

将指定分支合并到当前分支：

```shell
git merge <branch-name>
```

Git 会根据情况选择合并策略：

- **快进合并（Fast-forward）**：如果当前分支是目标分支的直接祖先，Git 只需移动指针，不会创建新的提交。
- **三方合并（Three-way merge）**：如果两个分支有分叉，Git 会使用两个分支的末端快照和它们的共同祖先进行三方合并，并创建一个新的合并提交。

#### **解决合并冲突**

当两个分支修改了同一个文件的同一部分时，Git 无法自动合并，会产生冲突：

```
<<<<<<< HEAD
当前分支的内容
=======
要合并的分支的内容
>>>>>>> branch-name
```

解决冲突的流程：

1. 打开冲突文件，手动编辑选择保留哪些内容
2. 删除冲突标记（`<<<<<<<`、`=======`、`>>>>>>>`）
3. `git add <file>` 将解决后的文件标记为已解决
4. `git commit` 完成合并提交

---

### **变基（Rebase）**

`rebase` 是另一种整合分支的方式。它会将当前分支的提交"重放"到目标分支之上，产生更线性的提交历史：

```shell
# 将当前分支变基到 main
git rebase main

# 交互式变基（修改、合并、删除提交）
git rebase -i HEAD~3
```

交互式变基中常用的操作：

| 命令 | 说明 |
|---|---|
| `pick` | 保留该提交 |
| `reword` | 保留提交但修改提交信息 |
| `squash` | 将该提交与前一个提交合并 |
| `drop` | 丢弃该提交 |

> **黄金法则：不要对已经推送到远程仓库的提交进行变基。** 变基会重写提交历史，如果其他人基于这些提交进行了开发，会造成混乱。

---

### **分支工作流**

#### **长期分支**

许多项目维护两个长期分支：

- `main`：始终保持稳定的可发布状态
- `develop`：用于日常开发和集成测试

#### **功能分支**

为每个新功能创建一个短期分支，开发完成后合并到 `develop` 或 `main`：

```shell
git switch -c feature/user-auth
# ... 开发功能 ...
git switch main
git merge feature/user-auth
git branch -d feature/user-auth
```

---

## **远程分支**

### **跟踪分支**

克隆仓库时，Git 会自动创建跟踪远程分支的本地分支。远程分支以 `<remote>/<branch>` 的形式命名，例如 `origin/main`。

```shell
# 查看远程分支
git branch -r

# 基于远程分支创建本地分支并跟踪
git checkout -b <branch> origin/<branch>
# 或者简写
git checkout --track origin/<branch>

# 设置已有本地分支跟踪远程分支
git branch -u origin/<branch>

# 查看所有跟踪分支的信息
git branch -vv
```

### **推送与拉取**

```shell
# 推送本地分支到远程
git push origin <branch>

# 推送并设置上游跟踪
git push -u origin <branch>

# 删除远程分支
git push origin --delete <branch>

# 拉取远程更新（推荐先 fetch 再 merge）
git fetch origin
git merge origin/main

# 或者直接 pull（fetch + merge）
git pull
```

---

## **Git 工具**

### **贮藏（Stash）**

当你正在开发某个功能但需要临时切换分支时，可以使用 `stash` 将未完成的修改保存起来：

```shell
# 贮藏当前修改
git stash

# 贮藏并添加说明
git stash push -m "work in progress: login page"

# 查看贮藏列表
git stash list

# 恢复最近一次贮藏
git stash pop        # 恢复并删除贮藏记录
git stash apply      # 恢复但保留贮藏记录

# 恢复指定的贮藏
git stash apply stash@{2}

# 删除贮藏
git stash drop stash@{0}
git stash clear      # 清除所有贮藏
```

---

### **Cherry-pick**

从其他分支选取特定的提交应用到当前分支：

```shell
git cherry-pick <commit-hash>

# 选取多个提交
git cherry-pick <hash1> <hash2>

# 选取一个范围内的提交
git cherry-pick <start-hash>..<end-hash>
```

---

### **二分查找（Bisect）**

当你发现了一个 bug 但不知道它是在哪次提交引入的，可以使用 `bisect` 通过二分法快速定位：

```shell
git bisect start
git bisect bad              # 标记当前提交为有 bug
git bisect good <commit>    # 标记一个已知没有 bug 的提交

# Git 会自动检出中间的提交，你需要测试并标记
git bisect good             # 这个提交没有 bug
git bisect bad              # 这个提交有 bug
# ... 重复直到找到引入 bug 的提交 ...

git bisect reset            # 结束查找，回到原来的分支
```

---

### **子模块（Submodule）**

子模块允许你在一个 Git 仓库中引用另一个 Git 仓库：

```shell
# 添加子模块
git submodule add <url> <path>

# 克隆包含子模块的仓库
git clone --recurse-submodules <url>

# 如果已经克隆但没有初始化子模块
git submodule init
git submodule update

# 更新所有子模块到最新
git submodule update --remote
```

---

### **搜索**

```shell
# 在工作目录中搜索字符串
git grep "pattern"
git grep -n "pattern"        # 显示行号
git grep --count "pattern"   # 显示匹配次数

# 在提交历史中搜索
git log -S "function_name"   # 搜索添加或删除了该字符串的提交
git log --all --grep="fix"   # 搜索提交信息
```

---

### **重写历史**

```shell
# 修改最后一次提交
git commit --amend

# 交互式变基修改多个提交
git rebase -i HEAD~3

# 将一个文件从所有历史中移除（谨慎使用）
git filter-branch --tree-filter 'rm -f <file>' HEAD
```

> **警告：** 重写历史会改变提交哈希。如果这些提交已推送到远程仓库，需要强制推送 `git push --force`，并通知所有协作者。

---

## **自定义 Git**

### **常用配置**

```shell
# 设置默认编辑器
git config --global core.editor "code --wait"

# 设置默认分支名
git config --global init.defaultBranch main

# 设置换行符处理
git config --global core.autocrlf true    # Windows
git config --global core.autocrlf input   # macOS/Linux

# 启用颜色输出
git config --global color.ui auto

# 设置 pull 默认使用 rebase
git config --global pull.rebase true

# 设置合并工具
git config --global merge.tool vscode
```

---

### **Git 钩子（Hooks）**

Git 钩子是在特定事件发生时自动执行的脚本，存放在 `.git/hooks/` 目录下。

常用的钩子：

| 钩子 | 触发时机 | 常见用途 |
|---|---|---|
| `pre-commit` | 提交前 | 代码检查、格式化 |
| `commit-msg` | 编写提交信息后 | 检查提交信息格式 |
| `pre-push` | 推送前 | 运行测试 |
| `post-merge` | 合并后 | 安装依赖 |

一个 `pre-commit` 钩子示例：

```bash
#!/bin/sh
# .git/hooks/pre-commit

# 运行代码格式检查
npm run lint
if [ $? -ne 0 ]; then
    echo "Lint 检查未通过，请修复后再提交。"
    exit 1
fi
```

> 推荐使用 `husky` 等工具来管理 Git 钩子，方便团队共享钩子配置。

---

## **GitHub 工作流**

### **Fork 与 Pull Request**

这是 GitHub 上最常见的协作模式：

1. **Fork** 目标仓库到自己的账户下
2. **Clone** 自己的 fork 到本地
3. 创建**功能分支**进行开发
4. **Push** 到自己的 fork
5. 在 GitHub 上创建 **Pull Request**（PR）

```shell
# 克隆自己的 fork
git clone https://github.com/your-name/project.git

# 添加上游仓库
git remote add upstream https://github.com/original/project.git

# 同步上游的最新代码
git fetch upstream
git merge upstream/main

# 创建功能分支，开发完成后推送
git switch -c feature/my-feature
# ... 开发 ...
git push origin feature/my-feature
# 然后在 GitHub 上创建 PR
```

---

### **SSH 密钥配置**

使用 SSH 协议可以免去每次推送时输入密码：

```shell
# 生成 SSH 密钥
ssh-keygen -t ed25519 -C "your_email@example.com"

# 将公钥添加到 GitHub
# 复制 ~/.ssh/id_ed25519.pub 的内容
# 在 GitHub Settings -> SSH and GPG keys -> New SSH key 中粘贴

# 测试连接
ssh -T git@github.com
```

---

## **常见问题与技巧**

### **回退提交**

```shell
# 创建一个新的提交来撤销指定提交（安全，不改变历史）
git revert <commit-hash>

# 回退到指定提交（危险，会改变历史）
git reset --soft <commit>     # 保留修改在暂存区
git reset --mixed <commit>    # 保留修改在工作目录（默认）
git reset --hard <commit>     # 丢弃所有修改
```

### **找回丢失的提交**

即使执行了 `reset --hard`，提交也不会立即被删除。可以通过 `reflog` 找回：

```shell
git reflog
git reset --hard <commit-hash>
```

### **清理工作目录**

```shell
# 查看哪些文件会被清理
git clean -n

# 清理未跟踪的文件
git clean -f

# 清理未跟踪的文件和目录
git clean -fd
```

### **暂时忽略已跟踪文件的修改**

```shell
# 暂时忽略
git update-index --assume-unchanged <file>

# 恢复跟踪
git update-index --no-assume-unchanged <file>
```

---

## **总结**

Git 是现代软件开发中不可或缺的基础工具。它的分布式架构保证了代码的安全性和协作的灵活性，轻量级的分支模型使得并行开发和实验变得轻松自如。

掌握 Git 的核心价值在于：

- **版本安全**：每一次提交都是项目的一个快照，任何时候都可以回溯到历史状态，不必担心代码丢失
- **高效协作**：分支、合并、Pull Request 等机制让团队成员能够独立工作又无缝整合
- **代码审查**：通过 diff、log、blame 等工具，可以清晰地追踪每一行代码的来龙去脉
- **持续集成**：Git 与 CI/CD 工具的结合，使得自动化测试和部署成为可能

对于日常使用，熟练掌握 `add`、`commit`、`push`、`pull`、`branch`、`merge` 这几个核心命令就能覆盖绝大多数场景。随着项目复杂度的提升，再逐步学习 `rebase`、`stash`、`cherry-pick` 等进阶工具即可。
