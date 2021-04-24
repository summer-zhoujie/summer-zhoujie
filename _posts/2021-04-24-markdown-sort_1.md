---
layout:     post
title:      "十大排序算法(一),冒泡排序"
subtitle:   ""
date:       2021-04-24 13:16:00
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 图解思路

> 最大/小的数 `冒泡` 到最后面

![17400545-534a0b911f89ebbf.gif](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b6dc6e2e514c44b7b5243138c6cd2f35~tplv-k3u1fbpfcp-watermark.image)

假设数组长度 n
1. 比较相邻2位, 大的数交换到右边, 直到第 n 个数也完成交换
2. n 减一 (第1步使得最大的数`冒泡`到了最右边)
3. 继续 1~2, 直到 n = 1

# 实现

```java
        // n 轮
        for (int i = length - 1; i > 0; i--) {
            boolean isSwap = false;
            // 两两比较
            for (int j = 0; j < i; j++) {
                if (array[j] > array[j + 1]) {
                    doSwap(array, j, j + 1);
                    isSwap = true;
                }
            }

            if (!isSwap) {
                return;
            }
        }
```

# 时间复杂度

`O(n) ~ O(n^2)`

最差: 当数组混乱无序时

f(n) = n (n-1) / 2 = O(n^2)

最好: 当数组基本有序, 只需要一轮比对就行

f(n) = n = O(n)

# 空间复杂度

`O(1)`, 不需要分配多余的空间

# 完整代码

```java
public class BubbleSort {

    private static void sort(int[] array) {
        if (array == null || array.length == 1) {
            return;
        }
        int length = array.length;


        for (int i = length - 1; i > 0; i--) {
            boolean isSwap = false;
            for (int j = 0; j < i; j++) {
                if (array[j] > array[j + 1]) {
                    doSwap(array, j, j + 1);
                    isSwap = true;
                }
            }

            if (!isSwap) {
                return;
            }
        }
    }

    private static void doSwap(int[] array, int x, int y) {
        int t = array[x];
        array[x] = array[y];
        array[y] = t;
    }

    public static void main(String[] args) {
        int[] array = {111, 52, 77, 98, 36, 12, 13, 48};
        sort(array);
        System.out.println(arrayToString(array));
    }

    private static String arrayToString(int[] array) {
        StringBuilder builder = new StringBuilder();
        for (int t : array) {
            builder.append(t + " ");
        }
        return builder.toString();
    }
}
```