---
title: DNF on CentOS 7
date: 2022-11-16 17:24:52
tags:
- CentOS
---

# DNF on CentOS 7

在 `CentOS 7` 上大部分时候需要通过 `YUM` 安装软件，但有些软件只能使用 `DNF`，所以必须先安装 `DNF` 后，再安装其他软件。

## 安装 DNF

安装 `DNF` 之前需要先安装并启用 `epel-release`

```shell
# install epel-release
yum install epel-release

# install dnf
yum install dnf
```

## 使用 DNF 安装软件

以安装 [zoxide](https://github.com/ajeetdsouza/zoxide) 为例，在 `zoxide` `github` 仓库中描述的安装方式如下：

> dnf copr enable atim/zoxide
> dnf install zoxide

但在使用 `dnf copr enable atim/zoxide` 时提示 `copr command doesn't exist`，实际上 `copr` 是一个插件，通过它来启用本地或在线软件源，而安装的 `DNF` 并不自带该插件。

```shell
# install copr
dnf install 'dnf-command(copr)'

# enable repository
dnf copr enable atim/zoxide

# install zoxide
dnf install zoxide
```

## 参考文章

[CentOS 安装 DNF](https://blog.csdn.net/lyc0424/article/details/107696000)

[fedora dnf 命令](https://blog.csdn.net/weixin_38566313/article/details/77688610)

[在CentOS 8上使用DNF管理软件包](https://www.jianshu.com/p/64e12bea3d49)
