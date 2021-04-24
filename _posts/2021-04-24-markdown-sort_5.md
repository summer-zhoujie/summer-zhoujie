---
layout:     post
title:      "十大排序算法(四), 希尔排序"
subtitle:   ""
date:       2021-04-24 13:18:29
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 算法图解

> `选择` 一个最大/小的数, 排到最前面

![17400545-e3a784e614fca758.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7c3762d0622e4d5b8930030aa4aca1b3~tplv-k3u1fbpfcp-watermark.image)

假设数组长度为 n
1. i = 0 , 遍历 [i,n] 个数字, `选择` 一个最小的, 和 i 交换
2. i + 1
3. 重复 1 2 步骤, 直到 i == n

# 实现

```java
        int n = array.length;
        int curMin;

        for (int i = 0; i < n - 1; i++) {
            curMin = i;
            for (int j = i + 1; j < n; j++) {
                curMin = array[curMin] > array[j] ? j : curMin;
            }
            doSwap(array, i, curMin);
        }
```

# 时间复杂度

f(n) = n(n-1)/2 = O(n^2)

# 空间复杂度

O(1), 没有使用多余的空间

# 完整代码

```java
class SelectionSort {

    private static void sort(int[] array) {
        if (array == null || array.length == 1) {
            return;
        }

        int n = array.length;
        int curMin;

        for (int i = 0; i < n - 1; i++) {
            curMin = i;
            for (int j = i + 1; j < n; j++) {
                curMin = array[curMin] > array[j] ? j : curMin;
            }
            doSwap(array, i, curMin);
        }
    }

    private static void doSwap(int[] array, int x, int y) {
        int t = array[x];
        array[x] = array[y];
        array[y] = t;
    }

    public static void main(String[] args) {
        int[] array = {111, 522, 77, 98, 36, 12, 13, 48};
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

