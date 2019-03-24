---
title: Git 常用命令
type: categories
copyright: true
date: 2019-03-13 12:12:07
tags: Git
categories: Git
---

### Git常用命令总结

***

<!-- more -->

#### 创建版本库

```shell
git init # 初始化 git 仓库
git add <filename> # 将文件添加到暂存区
git commit -m "some notification" # 把文件提交到仓库
git status # 查看工作区状态
git diff # 查看修改内容
```

#### 版本回退

```shell
git log # 显示提交日志
git log --pretty=oneline # 简化提交日志信息的输出
git reset --hard HEAD^ # 将当前版本回退到上一个版本
git reset --hard HEAD^^ # 将当前版本回退到上上一个版本
git reset --hard HEAD~100 # 将当前版本往上回退一百个版本
git reset --hard <目标版本的版本号> # 将当前版本回退到指定版本
git reflog # 查看命令记录，可用于撤销版本回退
```

#### 撤销修改

```shell
git checkout -- <filename> # 丢弃对应文件在工作区中的修改
git reset HEAD <filename> # 撤销添加到暂存区的修改
```

#### 删除文件

```shell
git rm <filename> # 从版本库中删除文件
git checkout -- <filename> # 如果误删的话也可以用这个命令将文件恢复
```

#### 添加远程仓库

```shell
ssh-keygen -t rsa -C "youremail@example.com" # 创建SSH Key
git remote add origin git@github.com:UserId/RepositoryName.git # 关联远程仓库和本地仓库
git push -u origin master # 推送master分支，-u参数可以使远程master分支和本地master分支关联起来，关联以后可以去掉-u参数
```

#### 从远程库克隆

```shell
git clone <repo-address> # 克隆远程仓库
```

#### 创建与合并分支

```shell
git branch # 查看分支
git branch <name> # 创建分支
git checkout <name> # 切换分支
git checkout -b <name> # 创建+切换分支
git merge <name> # 合并某分支到当前分支
git branch -d <name> # 删除分支
```

#### 解决冲突

```shell
git log --graph # 查看分支合并图
```

#### 分支管理策略

```shell
git merge --no-ff <name> # 禁用fast forward合并
```

#### Bug分支

```shell
git stash # 存储工作区
git stash list # 列出存储的工作区
git stash apply stash@{id} # 恢复与id对应的工作区，不会删除stash内容
git stash drop stash@{id} # 删除id对应的stash内容
git stash pop # 恢复的同时将stash内容也删除，类似于出栈
```

#### Feature分支

```shell
git branch -D <name> # 强行删除未合并的分支
```

#### 多人协作

```shell
git remote -v # 查看远程库信息
git push origin branch-name # 从本地推送分支
git pull # 从远程库拉取新的提交到本地，如果有冲突，要先处理冲突
git checkout -b branch-name origin/branch-name # 在本地创建和远程分支对应的分支
git branch --set-upstream branch-name origin/branch-name # 建立本地分支和远程分支的关联
```

#### Rebase

```shell
git rebase # 把本地未push的分叉提交历史整理成直线，这可以使得我们在查看历史提交的变化时更容易，因为分叉的提交需要三方对比
```

#### 创建标签

```shell
git tag <name> (<commit id>)# 新建一个标签，默认为HEAD，也可以指定一个commit id
git tag # 查看所有标签
git show <tag-name> # 查看标签信息
git tag -a <tag-name> -m "!@%!@" <commit id> # 创建带有说明的标签
```

#### 操作标签

```shell
git tag -d <tag-name> # 删除标签
git push origin <tag-name> # 推送某个标签到远程
git push origin --tags # 推送全部标签
git push origin :refs/tags/<tag-name> # 删除远程标签
```

#### 使用码云

```shell
git remote add origin git@gitee.com:UserId/RepositoryName.git # 关联码云的远程库，但如果你已经添加了origin，就会失败，需要先删掉已有的，再添加
###############################################
# 如果要同时关联两个远程库的话，就像下面这样
###############################################
git remote rm origin # 删除已有的origin远程库
git remote add github git@github.com:UserId/RepositoryName.git # 关联github
git remote add gitee git@gitee.com:UserId/RepositoryName.git # 关联gitee
git push github master # 将master分支推送到github
git push gitee master # 将master分支推送到gitee
```

#### 自定义git

```shell
git config --global color.ui true # 让git显示颜色
```

#### .gitignore

```shell
# 编写gitignore可以忽略文件
git add -f <file-name> # 强行添加被忽略的文件
git check-ignore -v <file-name> # 检查某文件被哪一条规则忽略了
```

#### 配置别名

```shell
git config --global alias.<short-name> <command-name> # 配置命令的别名
## 加了--global则对当前用户起作用，不加的话就只对当前仓库起作用
### 用户的配置文件在~/.gitconfig
### 仓库的配置文件在${repo-dir}/.git/config
## 修改这些文件就可以对别名进行增删
```

