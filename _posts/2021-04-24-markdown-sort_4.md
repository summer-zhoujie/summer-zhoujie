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

# 简介

`插入`排序的优化, 也叫缩小增量排序

# 算法思路

分组排序
> `步长`, 同组中2个元素在数组中的间隔数

1. 定义分组的步长, step = arrLength / 2
2. 对每个分组进行插入排序
3. step = step / 2
4. 重复 1~2 直到 step <= 0 (step == 1, 是最后一次分组)

第一次分组, step = 4

![8E467F87-936F-46F2-9609-4B26B74B46F1.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c0ffe65bf73941aeb09ad807da46a563~tplv-k3u1fbpfcp-watermark.image)

第一次分组, step = 2

![F4339AC7-63AD-403E-AADC-2722488FD050.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/347abdffd51e440a94125e4d98e817d5~tplv-k3u1fbpfcp-watermark.image)

第一次分组, step = 1

![072F7A87-BD31-4B5F-9FA4-43A5DA2FD5E9.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3273492bb0654765bcc24f03491931cf~tplv-k3u1fbpfcp-watermark.image)

```java
    private static void sort(int[] array) {
        if (array == null || array.length <= 1) {
            return;
        }

        int length = array.length;
        int step = length / 2;
        // step 为步长, 每次缩减为原来的 1/2
        for (; step > 0; step /= 2) {

            // 一共有step个序列
            for (int i = 0; i < step; i++) {

                // 对单个序列进行排序
                for (int j = i; j < length; j += step) {

                    int value = array[j];
                    int pre = j - step;
                    while (pre >= 0 && array[pre] > value) {
                        array[pre + step] = array[pre];
                        pre -= step;
                    }

                    array[pre + step] = value;

                }
            }
        }
    }
```

# 时间复杂度

> 参考
> * [Starrk的小屋-希尔排序简介](https://www.starrk.me/2020/04/01/%E6%8E%92%E5%BA%8F%E7%AE%97%E6%B3%95-(%E4%B8%8B)/)
> * [维基百科](https://zh.wikipedia.org/wiki/%E5%B8%8C%E5%B0%94%E6%8E%92%E5%BA%8F)

算法的时间复杂度和步长的定义相关

![468e800e78c97d073973e76acfe24962](希尔排序.resources/截屏2021-04-19 下午3.52.33.png)


# 空间复杂度

`O(1)`, 和`插入排序`一样, 并没有使用到对于空间


# 完整代码

```java
class ShellSort {
    private static void sort(int[] array) {
        if (array == null || array.length <= 1) {
            return;
        }

        int length = array.length;
        int step = length / 2;
        // step 为步长, 每次缩减为原来的 1/2
        for (; step > 0; step /= 2) {

            // 一共有step个序列
            for (int i = 0; i < step; i++) {

                // 对单个序列进行排序
                for (int j = i; j < length; j += step) {

                    int value = array[j];
                    int pre = j - step;
                    while (pre >= 0 && array[pre] > value) {
                        array[pre + step] = array[pre];
                        pre -= step;
                    }

                    array[pre + step] = value;

                }
            }
        }
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

