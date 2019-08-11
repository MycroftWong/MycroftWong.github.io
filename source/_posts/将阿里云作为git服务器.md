---
title: 将阿里云作为git服务器
date: 2019-08-11 15:42:57
categories: 
- 工具
  - git
tags:
- git
- aliyun
---

# 将阿里云作为git服务器

## 前言
本来一开始在研究，如何部署在`github`上的博客自动发布到`aliyun`上，这样就不用每次更新需要在`aliyun`服务器上进行一次`pull`操作

看了些使用`git hook`操作的文章，没有看到，很多都是一些步骤，也不说明原因，出了问题也不知道怎么解决。想到另外的方法是直接弄一个定时任务，这样也可以进行更新，不过感觉`git hook`的实现机制比较好。

不过忽然看到可以仓库直接放在`aliyun`上，`aliyun`提供了这样的功能，这也算是第一步吧。

## 流程

1. 在[aliyun code](https://code.aliyun.com)上添加本地电脑的公钥，建立安全的加密连接
2. 在[aliyun code](https://code.aliyun.com)上添加项目
3. 和常规仓库一样的操作

## 一、在[aliyun code](https://code.aliyun.com)上添加本地电脑的公钥，建立安全的加密连接

1. 查看本机电脑是否有公钥
公钥文件一般是`~/.ssh/id_rsa.pub`文件，同时对应一个`~/.ssh/id_rsa`的私钥文件

2. 生成公钥文件
使用命令`ssh-keygen -t rsa -C "1234567890@qq.com"`，其实使用`ssh-keygen`命令即可，`-t`参数指定算法，`-C`只是一个描述而已，官方建议是使用阿里云邮箱

3. 添加本地电脑的公钥，建立安全的加密连接

登陆网站[aliyun code](https://code.aliyun.com/)，选择`设置->SSH公钥->+增加SSH密钥->粘贴公钥的内容->增加密钥`

## 三、在[aliyun code](https://code.aliyun.com)上添加项目

登陆[aliyun code](https://code.aliyun.com)，选择`项目->新建项目->项目设置`，因为我的项目已经放在了`gitee`，所以选择了`其他仓库的链接`，这样直接就将项目导入了`aliyun code`，也可以新建一个项目（仓库），`pull`下来之后，将项目代码复制到该文件夹中。最后点击创建项目

## 四、修改`git`同时提交到多个仓库

原理就是增加远程仓库链接

1. 打开项目中的`./git/config`文件，其中的内容如下（我的项目）：
```config
[core]
	repositoryformatversion = 0
	filemode = false
	bare = false
	logallrefupdates = true
	symlinks = false
	ignorecase = true
[remote "origin"]
	url = https://gitee.com/mycroftwong/LoveServer.git
	fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
	remote = origin
	merge = refs/heads/master
```

可以看到它实际上就是绑定了一个远程仓库地址

2. 增加远程仓库地址
增加后的内容如下：
```config
[core]
	repositoryformatversion = 0
	filemode = false
	bare = false
	logallrefupdates = true
	symlinks = false
	ignorecase = true
[remote "origin"]
	url = https://gitee.com/mycroftwong/LoveServer.git
	fetch = +refs/heads/*:refs/remotes/origin/*
	url = https://code.aliyun.com/mycroftwong/LoveServer.git

[branch "master"]
	remote = origin
	merge = refs/heads/master
	
[remote "aliyun"]
    url = https://code.aliyun.com/mycroftwong/LoveServer.git
    fetch = +refs/heads/*:refs/remotes/gitee/*
    tagopt = --no-tags
```

3. 修改文件，进行`push`

如我修改了`README.md`文件
```git
# 添加文件
git add README.md
# 提交文件
git commit -m "修改README.md"
# 切换到本地master
git checkout master
# 将本地dev merge到master
git merge dev
# 将本地master push 到remote dev
git push origin master:dev


# 结果如下
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 4 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 322 bytes | 322.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
remote: Checking connectivity: 3, done.
remote: Powered By Gitee.com
To https://gitee.com/mycroftwong/LoveServer.git
   a8fe678..03d1b99  master -> dev
Enumerating objects: 5, done.
Counting objects: 100% (5/5), done.
Delta compression using up to 4 threads
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 322 bytes | 322.00 KiB/s, done.
Total 3 (delta 2), reused 0 (delta 0)
To https://code.aliyun.com/mycroftwong/LoveServer.git
   a8fe678..03d1b99  master -> dev
```

这样就自动提交到了多个仓库


## 参考文章
[使用阿里云code和git管理项目](https://blog.csdn.net/dark00800/article/details/54571859)

[git同时提交到2个仓库gitee github](https://blog.csdn.net/qq_34874784/article/details/89811192)

[git 实现关联 aliyun code 仓库](https://blog.csdn.net/woshidd0/article/details/81253913)

[使用阿里云作为git远程仓库的实践](https://blog.csdn.net/m0_37606574/article/details/80004680)

[4.3 服务器上的 Git - 生成 SSH 公钥](https://git-scm.com/book/zh/v2/%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E7%94%9F%E6%88%90-SSH-%E5%85%AC%E9%92%A5)

