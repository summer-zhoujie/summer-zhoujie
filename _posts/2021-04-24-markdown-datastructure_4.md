---
layout:     post
title:      "数据结构(四), 完美二叉树, 完全二叉树和完满二叉树"
subtitle:   ""
date:       2021-04-24 13:15:16
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 数据结构
    - 二叉树
---


# 完美二叉树 (Perfect Binary Tree)

A Perfect Binary Tree(PBT) is a tree with all leaf nodes at the same depth. All internal nodes have degree 2.

* 所有节点的度都是 2
* 所有叶子节点都在同一个层级

![4E9B6FC9-E9C2-4989-8D65-90FE89B673E0.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fe0160078b4244b3be88325e74ecfd88~tplv-k3u1fbpfcp-watermark.image)

> 度: 一个节点有几个孩子

# 完全二叉树 (Complete Binary Tree)

A Complete Binary Tree （CBT) is a binary tree in which every level,except possibly the last, is completely filled, and all nodes are as far left as possible.

* 除了最后一层外, 所有层都完美填充
* 最后一层所有叶子节点靠左对齐

![4A41A860-C35A-4D44-B03A-E3EA4F11AF1F.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6c8806a5faf4df49cd177b29e05ca60~tplv-k3u1fbpfcp-watermark.image)

# 完满二叉树 (Full Binary Tree)

A Full Binary Tree (FBT) is a tree in which every node other than the leaves has two children.

* 除去叶子节点, 所有节点的度都是 2

![A3379524-DB06-4B0E-A45A-BC988E4BF9FC.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ce154151eb8b40da9704f5a0db98b0c2~tplv-k3u1fbpfcp-watermark.image)

# 参考

[https://www.cnblogs.com/idorax/p/6441043.html](https://www.cnblogs.com/idorax/p/6441043.html)