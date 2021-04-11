---
layout:     post
title:      "数据结构(一), 二叉查找树BST"
subtitle:   ""
date:       2021-04-11 18:29:46
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 数据结构
    - 二叉树
---


# 一、别名
二叉搜索树, 有序二叉树, 排序二叉树, Binary Search Tree


# 二、特征
* 左子树的所有节点的值均小于根节点
* 右子树下所有节点的值均大于更节点
* 所有节点的值都不相同
* 任意节点的左子树和右子树也都是BST

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411181220196.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI1Nzc4MzY5,size_16,color_FFFFFF,t_70)

# 三、节点结构

```java
    public static class Node {
        // 数据区
        private int data;
        // 左节点
        private Node left;
        // 右节点
        private Node right;
    }
```

# 四、实现概述

## 1. 查找

* 先查找根节点, 
* `< 根`, 则找左子树; 
* `> 根`, 则找右子树; 
* `= 根`, 则找到返回;

算法时间复杂度 对于 n 个节点的树

* `最优` f(n) = 需要查找的次数 = 二叉树的层数 ~= O(logn)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411181237148.gif)


* `最差` f(n) = 需要查找的次数 = 二叉树的层数 = n = O(n)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411181253179.gif)

```java
    public Node search(int num) {
        return doSearch(root, num);
    }

    private Node doSearch(Node root, int num) {
        if (root == null) {
            return null;
        }

        if (root.data == num) {
            return root;
        } else if (root.data > num) {
            return doSearch(root.left, num);
        } else if (root.data < num) {
            return doSearch(root.right, num);
        }
        return null;
    }
```


## 2. 插入

* 比对根节点, 小于就往左节点比对, 大于就往右节点比对
* 直到需要比对的节点为空, 而这个空就是你需要插入的位置

算法时间复杂度: 

* `最优` f(n) = 需要比对的次数 = 二叉树的层数 ~= O(logn)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411181451238.gif)


* `最差` f(n) = 需要查找的次数 = 二叉树的层数 = n = O(n)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411181436660.gif)

```java
    public void insert(int num) {
        root = doInsert(root, num);
    }

    private Node doInsert(Node parent, int num) {
        if (parent == null) {
            parent = new Node(num);
        } else if (num > parent.data) {
            parent.right = doInsert(parent.right, num);
        } else if (num < parent.data) {
            parent.left = doInsert(parent.left, num);
        }

        return parent;
    }
```

## 3. 删除

* 先查找到目标节点
* 若: 目标左子树为空, 则, 用目标右子树根节点替换目标
* 若: 目标右子树为空, 则, 用目标左子树根节点替换目标
* 若: 都不为空, 则, 选取`左子树值最大节点`或者`右子树最小节点`替换目标, 并, 递归删除替换目标的节点

```java
    public void remove(int num) {
        root = doRemove(root, num);
    }

    private Node doRemove(Node parent, int num) {

        if (parent == null) {
            return null;
        }

        if (num > parent.data) {
            parent.right = doRemove(parent.right, num);
        } else if (num < parent.data) {
            parent.left = doRemove(parent.left, num);
        }
        // 找出左子树最大的值或者右子树最小的值替换, 这里选择前者来实现
        else if (parent.left != null && parent.right != null) {

            // 找到左子树最大值替换
            parent.data = findMax(parent.left).data;
            // 删除左子树中用于替换的节点
            parent.left = doRemove(parent.left, parent.data);
        }
        // 左子树为空, 直接用右子树根节点替换被删除的节点
        else if (parent.left == null) {
            parent = parent.right;
        }
        // 右子树为空, 直接用左子树根节点替换被删除的节点
        else if (parent.right == null) {
            parent = parent.left;
        }

        return parent;
    }

    private Node findMax(Node node) {
        if (node == null) {
            return node;
        }
        while (node.right != null) {
            node = node.right;
        }
        return node;
    }
```

算法复杂度:

* `最优` f(n) = 需要比对的次数 = 查找到目标比对次数 + 递归查找`替换目标的节点`的替换节点的比对次数 = 二叉树层数 = O(logn)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411181347169.gif)

* `最差` f(n) = ... = 二叉树层数 = n = O(n)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210411181407701.gif)


# 具体实现

就一个BinarySearchTree.java文件搞定, 里面还附有main()函数测试功能, 可直接运行[github传送门](https://github.com/summer-zhoujie/ZJPlayGround/blob/master/app/src/main/java/com/example/playground/binarytree/BinarySearchTree.java)