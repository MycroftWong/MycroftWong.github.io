---
title: 使用hexo搭建博客
date: 2019-08-09 11:21:43
tags:
---

# 使用hexo搭建博客

## 前言
之前也使用了`hexo`搭建博客，发布到`github`, 一开始发布了2篇文章之后也没怎么关心，一个是自己懒，二个是当时并不知道如果使用多用户管理，无法在公司的电脑上也同步更新，如果每次在公司写好，再回家整理发布，就有些太花时间了。所以专门抽空，花了几个小时学习了`git`分支管理

## 目标
1. 使用`hexo`搭建博客网站
2. 将博客发布到`gitee`上
3. 在多台电脑上管理
4. 将博客同步到个人服务器

## 一、使用`hexo`搭建博客网站

查看[hexo官方网站](https://hexo.io/zh-cn/)

### 1. 安装`git`
从[git官网](https://git-scm.com/)下载并安装`git`

安装完成后，使用命令`git version`查看版本，能查看到表示已经安装

### 2. 安装`node.js`

从[node.js官网](https://nodejs.org/en/)下载安装

使用命令`node -v`, `npm -v`查看版本

### 3. 安装`hexo`

使用命令`npm install -g hexo-cli`下载`hexo-cli`（`hexo`客户端）

### 4. 建立博客

具体如何使用`hexo`建立博客，查看`hexo`官网，这里说一下简单使用

使用命令`hexo init blog`在一个`blog`文件中建立博客，`blog`一定需要是空文件夹，进入文件夹之后，使用命令`hexo s`就启动了，然后查看网站`http://localhost:4000`即可查看到网站

### 5. 网站配置

查看`hexo`官网教程

### 6. 配置主题`next`

很简单，查看[next官网](http://theme-next.iissnan.com/)

## 二、将博客发布到`gitee`上

发布`gitee`博客非常简单，难点在于如何将`hexo`博客站点整体配置到仓库中。

下面是发布博客，只需要三步，如下。

1) 在`gitee`上新建博客站点

创建和用户名同名的仓库，如我的`gitee`用户名是`mycroftwong`, 我新建了一个`mycroftwong`仓库，里面没有任何文件，或者只有一个`README.md`文件

然后选择 服务 -> Gitee Pages -> 选择部署分支master -> 部署 即可

2) 修改博客配置文件`_config.yml`
下面是我的配置
```yml
# URL
## If your site is put in a subdirectory, set url as 'http://yoursite.com/child' and root as '/child/'
url: https://mycroftwong.gitee.io/
root: /
permalink: :year/:month/:day/:title/
permalink_defaults:

deploy:
  type: git
  repo: https://gitee.com/mycroftwong/mycroftwong.git
  branch: master
```

3) 发布

使用命令`hexo d`即可将博客发布到`gitee`上，然后再在gitee pages中更新即可通过`https://mycroftwong.gitee.io`访问到


## 三、在多台电脑上管理

在多台电脑上管理博客的原理，就是将这个博客发布到`git`仓库

两种实现方式：
1. 将`hexo`和博客网站放在不同的`git`仓库
2. 将`hexo`和博客网站放在同一个`git`仓库的不同分支

各有利弊，放在不同的仓库方便管理，放在同一个仓库也不用在两个仓库中麻烦，个人倾向于放在不同的仓库中，但是这里使用的是第二种方式

### 1. 建立`hexo`博客分支

将上面提到的仓库`pull`到本地，使用命令`git branch hexo`新建分支`hexo`, 切换到分支`hexo`

**这里说一下，`hexo`只能`init`空文件夹，官方建议如有需要，将文件复制到指定文件夹**

所以我们将之前的`blog`文件夹中的内容复制到这个分支中，然后添加、提交。同时也建立同名的远程分支，命令`git push origin hexo:hexo`，这样就建立了远程`hexo`分支，并将本地`hexo`的内容`push`到了远程分支`hexo`中。

### 2. 发布博客

知道原理就很简单了，本地`hexo`分支对应远程`hexo`分支，`master`分支对应远程`master`分支，我们在`hexo`分支中管理`hexo`博客网站，`master`则是实际的博客文件。具体想怎么操作，都可以的。

## 四、将博客同步到个人服务器

这一步并没有做，不过实际上，可以直接将仓库`pull`到个人服务器，使用`nginx`反向代理

## 后话

在写这篇文章的过程中发现，`gitee`提供了另一种方式，可以在配置gitee pages时，选择部署目录，那么可以不用建立分支，直接将整个`hexo`项目发布到一个分支中，只将`public`目录作为博客部署

## 参考文章
[使用 Hexo + Github 或 Gitee 搭建个人博客](https://blog.csdn.net/weixin_43215250/article/details/94055392)

[Hexo+github个人博客搭建+异地管理](https://blog.csdn.net/zwx2445205419/article/details/66970640)

[在VSCode中使用码云(Gitee)进行代码管理](https://blog.csdn.net/watfe/article/details/79761741)

[基于Gitee+Hexo搭建个人博客](https://segmentfault.com/a/1190000018662692)

[hexo官方网站](https://hexo.io/zh-cn/)

[hexo d后 ERROR Deployer not found: git](https://blog.csdn.net/weixin_36401046/article/details/52940313)

[git无法pull仓库refusing to merge unrelated histories](https://blog.csdn.net/lindexi_gd/article/details/52554159)

[Hexo托管到Coding；Hexo同时部署到多个平台](http://laker.me/blog/2015/09/28/15_0928_hexo_to_coding/)

[Hexo server报错Cannot read property 'offset' of null解决方法](lanhuapp.com/web/#/item/project/board?pid=4cba2aed-b116-4375-a9ef-e5a065cb1ed6)