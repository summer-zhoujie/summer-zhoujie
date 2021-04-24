---
layout:     post
title:      "十大排序算法(三), 插入排序"
subtitle:   ""
date:       2021-04-24 13:17:46
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 算法图解

> `插入` 第 i(范围0~length)个数,到有序队列的合适位置

![17400545-311766e7ef5be50c.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d8509ccc63604c4aa050f14e577ff4f1~tplv-k3u1fbpfcp-watermark.image)

假设数组长度 n , i(代表有序数组的长度) = 0
1. 取第 i+1 个数, `插入`到长度 i 有序数组的合适位置
2. i+1
3. 重复 1~2, 直到 i = n

# 实现

```java
        int n = array.length;

        for (int i = 0; i < n - 1; i++) {
            int waitToInsert = array[i + 1];
            int pos = i + 1;
            while (pos > 0 && waitToInsert < array[pos - 1]) {
                // swap
                array[pos] = array[pos - 1];
                pos--;
            }
            array[pos] = waitToInsert;
        }
```

# 时间复杂度

`O(n) ~ O(n^2)`

最好: 数组是有序的, 只需要遍历一遍

f(n) = n = O(n)

最差: 数组是混乱的

f(n) = n (n-1) / 2 = O(n^2)

# 空间复杂度

`O(1)` 没有使用到多余的空间

