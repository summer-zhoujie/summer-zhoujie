---
layout:     post
title:      "十大排序算法(八), 计数排序"
subtitle:   ""
date:       2021-04-24 13:21:13
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 简介

用来排序0~100数字最好的算法, 可以在`基数排序`中更有效的排序范围较大的数组

# 算法实现

**思路图示**

![20190712143216563.gif](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a5c79c3aa948416f9b0e0d4c70825a60~tplv-k3u1fbpfcp-watermark.image)

**具体步骤**

1. `最大/小值`: 找出Arr(待排序的数组)中最大(max)和最小(min)的元素；
2. 创建计数数组C, 长度是 K (K = max - min + 1)
3. `计数`: 统计数组中每个值为i的元素出现的次数，存入数组C的第i项；
4. `累加` C中数据进行累加(c[i]=c[i]+c[i-1]), 累加结束 c[i] 则代表 数字(i + min) 最终存储下标
5. `填充` B：将每个元素`(i+min)`放在新数组B的第`C(i)`项，每放一个元素就将`C(i)减去1`。

**算法图示**

`最大/小值` , `计数` , `累加`

![截屏2021-04-20 上午11.03.38.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b2a63891c3b846259cd18e328feddcc3~tplv-k3u1fbpfcp-watermark.image)

`填充`

![截屏2021-04-20 上午11.03.42.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/4a3b0f775b46454d90300fffd444aed2~tplv-k3u1fbpfcp-watermark.image)

```java
public static int[] sort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return arr;
        }

        // 找出最大值和最小值
        int max = 0;
        int min = 0;
        for (int i : arr) {
            max = Math.max(max, i);
            min = Math.min(min, i);
        }

        int k = max - min + 1;
        int[] c = new int[k];

        // 计数
        int length = arr.length;
        for (int i = 0; i < length; i++) {
            c[arr[i] - min] = c[arr[i] - min] + 1;
        }

        // 求和
        for (int i = 1; i < k; i++) {
            c[i] = c[i] + c[i - 1];
        }

        // 回填
        int[] b = new int[length];
        for (int i = 0; i < arr.length; i++) {
            int num = arr[i];
            int index = c[num - min] - 1;
            c[num - min]--;
            b[index] = num;
        }

        return b;
    }
```

# 时间复杂度

`O(n+k)`

f(n) = n(最大/小) + n(计数) + k(求和) + n(回填) = 3n + k = O(n+k)

# 空间复杂度

`O(n+k)`

f(n) = n(数组B) + k(数组c) = O(n+k)

# 参考

[维基百科-计数排序](https://zh.wikipedia.org/wiki/%E8%AE%A1%E6%95%B0%E6%8E%92%E5%BA%8F)

