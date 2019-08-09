---
title: hexo添加category目录
date: 2019-08-09 15:07:53
categories: 网站搭建
tags:
---

# hexo添加category目录

一开始我以为直接在配置文件`_config.yml`中去掉注释`categories`即可，这样是不对的，还需要手动建立`categories`目录

## 新建categories目录
使用命令`hexo new page categories`

新建完成之后虽然可以看到`category`目录，但是里面没有任何东西，需要修改文件`categories/index.md`，在其中添加`type: "categories"`表示指定的是`categories`

## 参考文章

[hexo的Next创建categories](https://blog.csdn.net/lcyaiym/article/details/76762089)

[hexo博客出现“Cannot GET/xxxx”的错误](https://www.cnblogs.com/Sroot/p/6305938.html)

