---
title: hexo-bug解决方案
date: 2019-08-12 09:12:08
categories: 网站搭建
tags:
- hexo
- bug
- git
---

### 1. git clone下来的主题，没有添加到项目中
前提：一开始知道需要删除`.git`文件夹

问题：提交之后发现，远程仓库只有一个主题目录，其中的内容并没有提交，尝试进入主题目录，添加所有的文件
```git
# 进入文件夹
$ cd themes/hexo-theme-matery
# 尝试添加所有文件
$ git add *
fatal: in unpopulated submodule 'themes/hexo-theme-matery'
# 出现了unpopulated错误，是因为没有删除其中的git缓存
# 退到themes文件夹
cd ..
# 删除缓存
$ git rm -rf --cached hexo-theme-matery
rm 'themes/hexo-theme-matery'
# 重新添加
$ git add ./hexo-theme-matery/*
# 提交
$ git commit -m "添加漏掉hexo-theme-matery主题"
```
