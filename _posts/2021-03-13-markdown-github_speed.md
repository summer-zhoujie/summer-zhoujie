---
layout:     post
title:      "github clone 提速"
subtitle:   "github clone 准备搭载火箭~"
date:       2021-03-21 17:55:13
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - github
    - 代理
---

# 1. 修改 Hosts 文件
利用 [https://www.ipaddress.com/](https://www.ipaddress.com/ ) 链接查询以下三个链接的DNS解析地址
1. github.com
2. assets-cdn.github.com
3. github.global.ssl.fastly.net

打开系统的 Hosts 文件进行修改

 - windows
修改C:\Windows\System32\drivers\etc\hosts文件的权限，指定可写入：右击->hosts->属性->安全->编辑->点击Users->在Users的权限“写入”后面打勾。如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210321174757732.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)
右击->hosts->打开方式->选定记事本（或者你喜欢的编辑器）->在末尾处添加以下内容：

```java
199.232.69.194 github.global.ssl.fastly.Net
140.82.114.3 github.com
185.199.109.153 assets-cdn.github.com
```

 - mac

```bash
sudo vim /ect/hosts
```
文末添加如下内容
```java
199.232.69.194 github.global.ssl.fastly.Net
140.82.114.3 github.com
185.199.109.153 assets-cdn.github.com
```

**注意**: CDN 地址会因为地域, 时间而变化, 每个人都是不一样的, 而且要定时刷新, 否则可能会失效 ( 又回到原来低速的github了 )

# 2. 镜像下载
> 主要用来提速 git clone

我们将原本的 clone 地址中的 **github.com** 进行**替换为 github.com.cnpmjs.org**
```powershell
git clone https://github.com.cnpmjs.org/bumptech/glide.git
```
再**恢复 origin 远程仓库配置**
```powershell
 git remote rm origin
 git remote add origin https://github.com/bumptech/glide.git
```

# 参考
[https://cloud.tencent.com/developer/article/1686178](https://cloud.tencent.com/developer/article/1686178)



