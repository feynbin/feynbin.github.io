---
title: git
abbrlink: 69c3279c
date: 2024-10-10 09:13:41
updated:
tags:
categories:
keywords:
description:
top_img: false
cover:
highlight_shrink:
---

> 推荐学习官方文档`https://git-scm.com/book/zh/v2/`

### 起步

版本控制是一种记录一个或若干文件内容变化，以便将来查阅特定版本修订情况的系统。

在DVCS中，客户端并不只提取最新版本的文件快照，而是把代码仓库完整地镜像下来，包括完整的历史记录，因此，任何一处协同工作用的服务器发生故障，事后都可以用任何一个镜像出来的本地仓库恢复。因为每一次的克隆操作，实际上都是一次对代码仓库的完整备份。

更进一步，借助这类系统可以指定和若干不同的远端代码仓库进行交互。籍此，在同一个项目中能够分别和不同工作小组的人相互协作。

---

#### 安装

#### windows

我推荐使用`scoop`来管理windows下的软件。我在另一篇文章中介绍了这个工具.

可以直接使用命令安装`git`.

```powershell
scoop install git
```

#### Ubuntu

我不建议源代码安装，因为`git`会周期性推送安全更新。在Ubuntu中进行安装.

```shell
sudo apt install git
```

对于想要体验最新稳定版本，使用此PPA.

```shell
sudo add-apt-repository ppa:git-core/ppa # apt update; apt install git
```

#### 初次配置

初次运行`git`前需要进行配置。每台计算机只需要配置一次，程序升级时会保留配置信息。

`git`自带一个`git config`的工具来帮助设置控制Git的外观和行为的配置变量。这些变量存储在三个不同的位置。

1. `/etc/gitconfig`. 包含系统上每一个用户及他们仓库的通用配置。 如果在执行 git config 时带上 --system 选项，那么它就会读写该文件中的配置变量。 （由于它是系统配置文件，因此你需要管理员或超级用户权限来修改它。）
2. `~/.gitconfig`或者`~/.config/git/config`. 只针对当前用户。 你可以传递 `--global` 选项让 Git 读写此文件，这会对你系统上 **所有** 的仓库生效。
3. 当前使用仓库的Git目录中的`config`文件(即`.git/config`). 针对该仓库。 你可以传递 `--local` 选项让 Git 强制读写此文件，虽然默认情况下用的就是它。 （当然，你需要进入某个 Git 仓库中才能让该选项生效。）

每一个级别会覆盖上一个级别的配置，所以`.git/config`的配置变量会覆盖`/etc/gitconfig`中的配置变量。

在windows中，这些文件的位置分别位于

1. `C:\ProgramData\Git\config`(如果使用scoop安装，则配置文件目录被修正为scoop文件夹)

2. `C:\Users\$User\\.gitconfig`
3. 当前仓库中的`.git/config`

通过以下命令查询所有的配置以及它们所在的文件

```shell
git config --list --show-origin
```

#### 用户信息

设置你的用户名和邮件地址，每一个`git`提交都会使用这些信息，它们会写入到你的每一次提交中，不可更改。

```shell
git config --global user.name "username"
git config --global user.email "user@example.com"
```

你可以使用`git config --list`来列出所有Git当前能找到的配置。

### Git基础

只有在命令行模式下才能执行git的所有命令。

#### 获取Git仓库

通常有两种方式:

1. 将尚未进行版本控制的本地目录转换为Git仓库；
2. 从其他服务器克隆一个已存在的Git仓库.

在已存在目录中初始化仓库，于目录下运行:

```shell
git init
```

这将创建`.git`目录，包含初始化的Git仓库中所有的必须文件。

克隆现有的仓库:

```shell
git clone https://github.com/libgit2/libgit2
```

这将会在当前目录下创建名为`libgit2`的文件夹，并从远程仓库中拉取所有数据到该目录下的`.git`文件夹，然后从中读取最新版本的文件的拷贝。

希望在克隆远程仓库的时候自定义本地仓库的名字，你可以通过额外的参数指定:

```shell
git clone https://github.com/libgit2/libgit2 mylibgit
```

#### 记录更新

工作目录下的所有文件都有两种状态: **已跟踪**或**未跟踪**。

`git status`用于查看那些文件处于什么状态。

`git add file`用于精确地将内容添加到下一次提交中。可以用它开始跟踪新文件，或者把已跟踪的文件放到暂存区，还能用于合并时把有冲突的文件标记为已解决状态。

#### 忽略文件

总有些文件无需纳入Git的管理，也不希望它们总出现在未跟踪文件列表。

在这种情况下，我们可以创建一个名为`.gitignore`的文件，列出要忽略的文件的模式。

这个文件的格式规范如下:

- 所有空行或以`#`开头的行都会被Git忽略
- 可以使用标准的glob模式匹配，它会递归地应用在整个工作区中。
- 匹配模式可以以(`/`)开头防止递归。
- 匹配模式可以以(`/`)结尾指定目录。
- 要忽略指定模式以外的文件或目录，可以在模式前加上叹号(`!`)取反。

#### 查看修改

查看已暂存的修改:

`git diff`用于比较工作目录中当前文件和暂存区域快照之间的差异。

`git diff --staged`查看已暂存文件与最后一次提交的文件差异

#### 提交更新

`git commit`用于提交暂存区的内容，这个命令会启动文本编辑器来输入提交说明。

`git commit -a`这个命令会跳过使用暂存区域，将所有已经跟踪过的文件暂存起来一并提交，从而跳过`git add`步骤。

#### 移除文件

`git rm file`将file移除，在下一次提交时不再纳入版本管理。

`git rm --cached file`将文件从Git仓库中擅长，但仍然保留在当前工作目录中。

`git rm`命令支持**glob**模式

#### 移动文件

不像其他的VCS系统，Git并不显式跟踪文件移动操作。如果在Git中重命名了某个文件，仓库的元数据并不会体现这是一次改名操作。

`git mv file_from file_to`在Git中对文件改名。

这相当于执行了:

```shell
mv file_from file_to
git rm file_from
git add file_to
```

#### 查看提交历史

`git log`是简单而有效的工具，用于回顾提交历史。

不传入任何参数的默认情况下, `git log`会按照时间先后顺序列出所有的提交，最近的更新排在最上面。

这个命令有很多选项可以帮助搜寻需要的提交，`--patch`会显示每次提交所引入的差异。也可以限制显示的日志条目数量，例如使用`-2`选项来只显示最近的两次提交

### Git分支

### Git服务器

### 分布式

### Github

### 工具

### 自定义

### 其他

### 内部原理