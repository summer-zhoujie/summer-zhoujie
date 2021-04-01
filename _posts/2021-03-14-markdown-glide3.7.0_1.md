---
layout:     post
title:      "Android Glide 3.7.0 源码解析(一), 准备工作"
subtitle:   ""
date:       2021-03-14 15:02:28
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - glide3.7.0
    - 源码
---

# 一、前言

## 1. 关于源码阅读
- 不要妄图一下子窥得全貌, 一个开源项目是由很多人同时维护, 一个人不可能一下子掌握每个细节, 可以先从一个特定流程入手, 这样会使得源码阅读事半功倍.
- 阅读的过程中尽量不要过分纠结主线之外的细节, 可能会给整个主线的分析带来极大难度

## 2. 关于Glide的使用
> [官方文档中文版传送门](https://muyangmin.github.io/glide-docs-cn/)
> [官方文档英文版传送门](https://bumptech.github.io/glide/)
> [github地址](https://github.com/bumptech/glide)

# 二、源代码准备
> [Glide 3.7.0 源代码官方链接](https://github.com/bumptech/glide/tree/v3.7.0)
> [Glide github地址](https://github.com/bumptech/glide)

## 1. 下载
github 如果同步较慢点击[这里](https://summer-zhoujie.github.io/2021/03/13/markdown-github_speed/)

```powershell
git clone https://github.com/bumptech/glide.git
```
## 2. 切换到v3.7.0分支

```powershell
git checkout v3.7.0 -b v3.7.0
```
# 三、文章传送门

1. Android Glide 3.7.0 源码详解 (一) , 准备工作
2. [Android Glide 3.7.0 源码解析 (二) , 从一次图片加载流程看源码](/2021/03/14/markdown-glide3.7.0_2/index.html)
3. [Android Glide 3.7.0 源码解析(三), 生命周期绑定](/2021/03/20/markdown-glide3.7.0_3/index.html)
4. [Android Glide 3.7.0 源码解析(四), BitmapPool作用及原理](/2021/03/20/markdown-glide3.7.0_4/index.html)
5. [Android Glide 3.7.0 源码解析(五) , 如何获得ImageView的宽高](/2021/03/20/markdown-glide3.7.0_5/index.html)
5. [Android Glide 3.7.0 源码解析(六) , 缓存结构详述](/2021/03/27/markdown-glide3.7.0_6/index.html)
5. [Android Glide 3.7.0 源码解析(七) , 细说图形变换和解码](/2021/03/31/markdown-glide3.7.0_7/index.html)

# 四、参考

- Glide最全解析: [https://blog.csdn.net/guolin_blog/category_9268670.html](https://blog.csdn.net/guolin_blog/category_9268670.html)