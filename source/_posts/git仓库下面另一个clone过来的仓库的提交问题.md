---
title: git仓库下面另一个clone过来的仓库的提交问题
top: false
cover: false
toc: true
mathjax: true
date: 2022-05-07 13:58:53
password:
summary:
tags:
	- hexo
	- blog
	- 搭建
	- git
categories: hexo
---

如果你的 git 仓库下面还有另外的 clone 过来的仓库，那么在你正常的提交代码时`git commit`的时候一定会出现如下图的错误

```bash
D:\blog\nevan-blog> git commit -m 'commit'
On branch master
Your branch is a ahead of 'origin/master' by 1 commit.
  <use 'git push' to publish your local commit>
Changes not staged for commit:
		modified:	themes/matery(modified content)

no changes add to commit
D:\blog\nevan-blog>
```

并且上传到仓库的文件夹是空的

#### 1. 先强行删除 clone 来的目录下面的.git 文件

   删除方式：在该目录下打开命令行工具，执行`rd /s/q .git`
   删除成功后执行`ls .git`命令查看是否删除成功

#### 2. 回到仓库根目录删除仓库中的空文件夹

   - `git rm -r --cached "themes/matery"`
   - `git commit -m "remove empty folder"`
   - `git push origin master`

#### 3. 在仓库根目录删除仓库中的空文件夹

   - `git add .`
   - `git commit -m "remove empty folder"`
   - `git push origin master`

#### 4. 在仓库根目录重新提交代码

   - `git add .`
   - `git commit -m "repush"`
   - `git push origin master`

这样就能保证不报上面的错，并且删除了空文件夹，重新把 clone 下来的目录上传到仓库重

#### 说明下出现这种情况的原因：

> 由于 clone 下来的文件夹也是一个 clone 仓库，因此正常的`git add .`是无法提交改文件夹下的文件的，所以我们要做的就是删除文件夹下的.git 文件夹使其不关联 clone 的仓库，这样就能通过 `git add .`命令来提交内容了
