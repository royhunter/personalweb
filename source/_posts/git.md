title: 常用Git命令速查笔记
date: 2016-06-14 21:26:32
tags: Tools
categories: Programme
---
常用Git命令总结与速查笔记

<!--more--> 

参考文档: "Pro Git---Scott Chacon"


## Git基础操作
几幅有助于理解GIT操作流程的示意图：
![](http://7j1zal.com1.z0.glb.clouddn.com/git_full.PNG)
![](http://7j1zal.com1.z0.glb.clouddn.com/git_trans.PNG)

### 创建项目的Git仓库
#### 从当前目录初始化
```bash
$ git init                 将当前目录初始化为git仓库
$ git init [project-name]  新建一个目录，将其初始化为Git代码库    
```
初始化后，在当前目录下会出现一个名为.git的目录，所有Git需要的数据和资源都存放在这个目录中。

```bash
$ git add [file]              添加到暂存区
$ git commit -m 'comments'    check in到仓库
```

#### 从现有仓库clone
```bash
$ git clone git://github.com/schacon/grit.git    从现有仓库克隆
```

### 查看状态
```bash
$ git status
```

### 查看diff
```bash
$ git diff              查看当前文件（工作区）与暂存区的差异
$ git diff --cached     查看暂存文件与上次提交的差异
$ git diff HEAD         查看工作区与上次提交的差异
```

### 跳过使用暂存区域
```bash
$ git commit -a -m 'comments'
```

### 移除文件
```bash
$ git rm [file]           从仓库移除，同时也从本地文件夹删除
$ git rm --cached [file]  从仓库移除，但不从本地文件夹删除
```

### 移动/重命名文件
```bash
$ git mv [file_from] [file_to]       
```

### 查看提交历史
```bash
$ git log            查看提交历史
$ git log --p -2     查看提交内容差异， 显示2个 
```

### 撤销操作
#### 修改最后一次提交
```bash
$ git add [forgotten_file]    补上暂存操作
$ git commit --amend          运行 --amend 提交
```

#### 撤销暂存区的文件（即撤销已经git add操作）
```bash
$ git reset HEAD <file>
```

#### 取消对文件的修改
```bash
$ git checkout -- <file>
```

#### 在历史版本之间切换
```bash
$ git reflog          查看命令历史， 可以显示历史commit id
$ git reset --hard [commit_id]
```

## Git 分支

![](http://7j1zal.com1.z0.glb.clouddn.com/branch_1.PNG)
![](http://7j1zal.com1.z0.glb.clouddn.com/branch_2.PNG)

### 查看分支信息
```bash
$ git branch      list所有本地分支
$ git branch -r   list所有远程分支
$ git branch -a   list所有分支
```

### 新建分支
```bash
$ git branch [branch-name]    新建一个分支，但依然停留在当前分支
$ git checkout -b [branch]    新建一个分支, 并切换到该分支
```

### 切换分支
```bash
$ git checkout [branch-name]    切换到指定分支
```

### 合并分支
```bash
$ git merge [branch]     合并指定分支到当前分支
```

### 删除分支
```bash
$ git branch -d [branch-name]     删除指定分支
```

### 解决冲突
```bash
$ git status           查看文件冲突信息
$ git add [file_name]  将冲突文件标记为解决
$ git mergetool        使用可视化的合并工具
```

## 远程仓库的使用
### 显示远程仓库
```bash
$ git remote      显示仓库名
$ git remote -v   显示详细信息
```

### 添加远程仓库
```bash
$ git remote add [shortname] [url]
```

### 从远程仓库抓取数据
```bash
$ git fetch [remote-name] 
```
此命令会到远程仓库中拉取所有你本地仓库中还没有的数据.
需要记住，fetch 命令只是将远端的数据拉到本地仓库，并不自动合并到当前工作分支，只有当你确实准备好了，才能手工合并。

```bash
$ git pull [remote] [branch]   取回远程仓库的变化，并与本地分支合并
```

### 推送数据到远程仓库
```bash
$ git push [remote-name] [branch-name]  上传本地指定分支到远程仓库
```

### 查看远程仓库信息
```bash
$ git remote show [remote-name]  查看某个远程仓库的详细信息
```

### 跟踪远程分支
```bash
$ git checkout -b [分支名] [远程名]/[分支名]
```

### 删除远程分支
```bash
$ git push [remote-name] :[branch-name]
```

## 标签操作
### 列显已有的标签
```bash
$ git tag
```

### 新建标签
#### 含附注的标签
```bash
$ git tag -a [tag_name] -m 'comments'
```

#### 轻量级标签
```bash
$ git tag [tag_name]
```

### 查看标签信息
```bash
$ git show [tag_name]
```

### 推送标签到远程仓库
```bash
$ git push [remote] [tag_name]
```

### 新建一个分支，指向某个tag
```bash
git checkout -b [branch] [tag_name]
```

## 学习网站
[https://git-scm.com/book/zh/v2](https://git-scm.com/book/zh/v2)
[https://git.wiki.kernel.org/index.php/Main_Page](https://git.wiki.kernel.org/index.php/Main_Page)
[http://gitready.com/](http://gitready.com/)
[https://www.kernel.org/pub/software/scm/git/docs/](https://www.kernel.org/pub/software/scm/git/docs/)