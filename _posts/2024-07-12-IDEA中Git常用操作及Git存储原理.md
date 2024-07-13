---
title: IDEA中Git常用操作及Git存储原理
categories: [ 编程, Git ]
tags: [ git ]
---

## Git简介与使用

### Intro

> Git is a free and open source **distributed version control system** designed to handle everything from small to very
> large projects with speed and efficiency.

Git是一款分布式版本控制系统（VCS），是团队合作开发的必备工具。

Git Repository 本地可划分为3个区域：

- workspace
- staging area
- local repository

其交互示意图如下：
![](/assets/2024/07/12/git.png)

git基本的命令操作可以参看：[Git教程](https://www.liaoxuefeng.com/wiki/896043488029600)

### Using within Jetbrains IDE

Jetbrains全家桶提供了对Git的支持，下面以`IDEA 2024.1.4`版本演示Git基本操作。

IDEA使用不同的颜色表明文件在Git中的状态，如下所示为Darcula主题下颜色的含义：

![](/assets/2024/07/12/git_idea_color.png)

具体颜色含义可参看：[Check file status](https://www.jetbrains.com/help/idea/adding-files-to-version-control.html#a8dcy8_284)

#### rollback

如果我本地改动了文件，但发现改动无效，想回到改动之前的文件状态，可以`右击文件 -> Git -> Rollback`:
![](/assets/2024/07/12/rollback.png)

#### commit & amend commit & commit checks

使用IDEA commit窗口可以方便地选择需要commit的文件，填写commit信息。此外如果上一次commit有东西忘写了，这次写完使用amend
commit将上次和这次改动合并成一次commit，在commit窗口勾选amend即可。

![](/assets/2024/07/12/git_idea_color.png)

每次commit前需要有些前置校验呢？比如自动格式化代码、自动checkstyle，IDEA也提供了该项功能：在commit窗口，点击设置图标，勾选所需checks

| 1.设置                                           | 2.勾选                                            |
|------------------------------------------------|-------------------------------------------------|
| ![](/assets/2024/07/12/git_idea_gitchecks.png) | ![](/assets/2024/07/12/git_idea_gitchecks2.png) |

#### squash commit

如果我们要将多次提交合并成一个提交，可以定位到当前本地分支，并多选多条commit进行squash：

![](/assets/2024/07/12/git_idea_squash.png)

此时push就需要强制push了

#### rebase

当master代码有变动时，基于老master版本开发的本地feature分支需要rebase到最新代码（即把master分支新代码合并到本地feature开发commit之前），使用IDEA操作：

1. 拉取master最新代码 git pull
2. 切换到feature分支
3. 执行rebase

| 1. 拉取master最新代码                             | 2.执行rebase                                  |
|---------------------------------------------|---------------------------------------------|
| ![](/assets/2024/07/12/git_idea_update.png) | ![](/assets/2024/07/12/git_idea_rebase.png) |

#### cherry-pick

合作开发时，比如A同学在`feature-a`分支开发功能，B同学在`feature-b`分支开发，且`feature-b`分支依赖`feature-a`分支的功能，即`feature-b`分支基于`feature-a`分支的commit开发。
如果`feature-a`分支的commit有改动，则`feature-b`就要同步这些改动，且需要保证`feature-b`分支的commit不会凌乱，这时可使用cherry-pick来完成。

1. 拉取`feature-a`最新代码；
2. 复制分支`feature-b`到`feature-b-bak`；
3. 对`feature-b`执行`reset --hard`到`feature-a`的最早commit之前的一次commit；
4. 在`feature-b`中`cherry-pick` 新的`feature-a`的commit
5. 在`feature-b`中`cherry-pick` 复制的`feature-b-bak`的自己的commit

| `reset --hard`                             | `cherry-pick`                                   |
|--------------------------------------------|-------------------------------------------------|
| ![](/assets/2024/07/12/git_idea_reset.png) | ![](/assets/2024/07/12/git_idea_cherrypick.png) |

#### ref

- [Boost Your Productivity: 13 Pro Git Tips for JetBrains IDEs](https://www.youtube.com/watch?v=vCGCgxTet1U&t=2721s)

## Git存储原理

git中有三种类型的文件：
- blob: 压缩存储的二进制文件内容
- tree: 表示项目文件夹，tree下包含subtree和blob，以及blob对应文件的文件名、访问权限等信息，使用Merkel Hash Tree数据结构构建
- commit: 包含指向的tree和提交信息


3种类型文件均存储于`./.git/objects/`文件夹下，利用40位的SHA的hash值的前2位作为文件夹，后38位作为文件名，其组织形式如下图：

![](/assets/2024/07/12/git_objects.png)

假设git项目文件夹下有一个a.txt文件，执行完`git add .`和`git commit -m "first commit"`后`./.git/objects/`文件夹新增3个objects：
```bash

> watch -n .5 tree .git
...
├── objects
│    ├── 08
│    │    └── 585692ce06452da6f82ae66b90d98b55536fca
│    ├── 47
│    │    └── d94322168b96993a93f2346c8eafd50bcc8317
│    ├── 78
│    │    └── 981922613b2afb6025042ff6bd878ac1994e85
|...

```
而分支是指向commit hash的一个引用，其存储在 `.git/refs/heads/`下：
```bash
> pwd
/Users/bty/IdeaProjects/testgit/.git/refs/heads
> ls
feature-a	main
> cat main 
47d94322168b96993a93f2346c8eafd50bcc8317
```

`HEAD`表示当前的分支，指向 `.git/refs/heads/`的一个文件：
```bash
> pwd
/Users/baotingyu/IdeaProjects/testgit/.git
> ls
COMMIT_EDITMSG	config		hooks		info		objects
HEAD		description	index		logs		refs
> cat HEAD 
ref: refs/heads/feature-a
```

objects文件类型可通过`git cat-file -t [40位hash]`查看：
```bash

> git cat-file -t 78981922613b2afb6025042ff6bd878ac1994e85
blob

> git cat-file -t 08585692ce06452da6f82ae66b90d98b55536fca
tree

> git cat-file -t 47d94322168b96993a93f2346c8eafd50bcc8317
commit

```

以上文件的内容均可以通过`git cat-file -p [40位hash]`：
```bash

# 查看一个blob，显示文件内容（不包含文件名，访问权限等信息，都在tree里面）
> git cat-file -p 78981922613b2afb6025042ff6bd878ac1994e85
a

# 查看一个tree，显示tree下的内容（子tree或blob）
> git cat-file -p 08585692ce06452da6f82ae66b90d98b55536fca
100644 blob 78981922613b2afb6025042ff6bd878ac1994e85	a.txt

# 查看一个commit，包含指向的tree和提交信息
> git cat-file -p 47d94322168b96993a93f2346c8eafd50bcc8317
tree 08585692ce06452da6f82ae66b90d98b55536fca
author bty <bty@com> 1720876267 +0800
committer bty <bty@com> 1720876267 +0800

first commit

```





