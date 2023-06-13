---
layout: install-cmake.md
title: Install CMake
date: 2023-06-13 15:08:01
tags:
---


# Install CMake

`Ubuntu 22.04` 仓库目前的 `CMake` 版本是 `3.22.1-1ubuntu1.22.04.1`，有时需要安装更新的版本，则可以下载官方包进行安装。

下载地址：

- https://github.com/Kitware/CMake/releases
- https://cmake.org/files/

如我们下载 `3.25.0` 的文件 `https://cmake.org/files/v3.25/cmake-3.25.0-linux-x86_64.sh`。

下载后执行以下命令，就可以将执行文件和相关的文档说明安装到 `/usr/local` 中：

```shell
bash cmake-3.25.0-linux-x86_64.sh --prefix=/usr/local --exclude-subdir
```
