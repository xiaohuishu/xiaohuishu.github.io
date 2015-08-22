---
layout: post
title: "Git常用命令说明"
date: 2015-01-22 11:14:55
categories: Git
excerpt: 基本的Git命令介绍,待更新...
---

* content
{:toc}

## 简介

Git是一个分布式版本控制系统,在此介绍Git基本常用命令

首先需安装Git请参照[廖雪峰Git教程][1]

  [1]:http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/00137396287703354d8c6c01c904c7d9ff056ae23da865a000

---

### Git常用命令使用

1.git init

> 可以使你在本地建立的一个文件夹变成Git可以管理的仓库(即版本库)
> 假设在本地建立一个myblog的文件夹

	antsmarth-/myblog$: git init
	Initialized emptty Git repository in /homt/antsmarth/myblog/.git/

>这时候版本库已经建立好了,并且在当前目录上多了一个.git的目录
>这个目录是用来追踪管理版本库

2.git add

> 将文件添加到仓库

	$: git add test.txt //将test.txt添加到仓库中但是操作的目录必须是你所建的Git仓库目录
	$: git add . //将目录下所有的文件添加到仓库

3.git commit
	
> 把添加的文件(add操作)提交到仓库

    $: git commit -m "add test.txt"

4.git status

> 此命令可以让我们时刻掌握仓库当前的状态,比如你添加test.txt到仓库之后修改了test.txt文件用此命令可以查看修改的状态

5.git diff

> 利用git status命令可以让我们知道我们修改了test.txt文件但是我们不知道修改了什么内容,git diff可以查询具体修改了什么内容

	$: git diff HEAD --test.txt查看test.txt工作区和版本库里最新版本的区别

6.git log

> 查看版本库的历史记录,显示最近到最远的提交日志

	git log会按提交时间列出所有的更新,最近的更新排在最上面.
	git log -p 会展开显示每次提交的内容差异,git log -p -2则显示最近的两次更新
	git log --pretty=oneline 输出简单的提交信息

详细介绍[Git教程之git log详解](http://git-scm.com/book/zh/v1/Git-%E5%9F%BA%E7%A1%80-%E6%9F%A5%E7%9C%8B%E6%8F%90%E4%BA%A4%E5%8E%86%E5%8F%B2)

7.git reset

> 回退命令,可以将当前版本回退到上一个版本或者指定的版本

	git reset --hard HEAD^ 回退到上一版本
	git reset --hard [commit id] 回退到指定的版本,commit id可以通过git log来查看

8.git rm [file]

> 删除版本库中的指定文件,若不小心删除则可以通过git checkout --[file]命令,此命令是用版本库的版本替代工作区的版本
	
9.git remote add origin git@github.com:xiaohuishu/myblog.git

> * (xiaohuishu为github用户名,myblog.git为github中的一个仓库)
> 此命令是将本地仓库与github中建立的仓库进行连接,连接之后可以将本地内容推送到github上

配置远程仓库[Git教程之配置远程仓库](http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001374385852170d9c7adf13c30429b9660d0eb689dd43a000)

10.git push origin [branch]

> * [branch]为分支名,比如master,gh-pages
>此命令是将本地的仓库内容推送到远程仓库上

11.git clone git@github.com:xiaohuishu/myblog.git

> 将你github中的myblog克隆到本地

---

### 分支管理待更新

---




