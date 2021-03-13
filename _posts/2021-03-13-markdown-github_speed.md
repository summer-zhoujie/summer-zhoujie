---
layout:     post
title:      "github clone 提速"
subtitle:   "github clone 准备搭载火箭~"
date:       2021-03-13 18:26:06
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - github
    - 代理
---

# 一、正常下载 Glide 源代码
```powershell
 git clone https://github.com/bumptech/glide.git
```
# 二、提速方案下载 Glide
## 1. 下载
我们将原本的网站中的 github.com 进行替换为 github.com.cnpmjs.org
```powershell
 git clone https://github.com.cnpmjs.org/bumptech/glide.git
```
## 2. 查看 origin 配置
```powershell
 git config --list
```
 发现:
 remote.origin.url=https://github.com.cnpmjs.org/bumptech/glide.git
## 3. 修改 origin 配置
```powershell
 git remote rm origin
 git remote add origin https://github.com/bumptech/glide.git
```


