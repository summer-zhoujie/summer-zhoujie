---
layout:     post
title:      "十大排序算法(十), 基数排序"
subtitle:   ""
date:       2021-04-24 13:22:58
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 算法图解

![20200429173859195.gif](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b7ff87606aac4db3bb9c37e0bb683947~tplv-k3u1fbpfcp-watermark.image)

1. 将数切割成个位, 十位, 百位, 千位...
2. 按位放置进入长度为10的数组
3. `先进先出`规则回填入数组
4. 重复1~3,直到所有位数都比较完毕

```java
    public static void sort(int[] arr) {

        if (arr == null || arr.length <= 1) {
            return;
        }

        // 初始化比较需要的`桶`空间
        ArrayList<ArrayList<Integer>> buckets = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            buckets.add(new ArrayList<>());
        }

        int radix = 1;
        // 循环比较各个位数
        while (true) {

            boolean isAllZero = true;

            // 按照位数放置到相应的桶位置
            for (int i = 0; i < arr.length; i++) {
                // 取出整数的个,十, 百, 千...
                int num = arr[i] / radix % 10;
                if (isAllZero && num != 0) {
                    isAllZero = false;
                }
                buckets.get(num).add(arr[i]);
            }

            if (isAllZero) {
                break;
            }

            // 回填Arr && 清空桶(buckets)中数据
            int index = 0;
            for (ArrayList<Integer> bucket : buckets) {
                while (!bucket.isEmpty()) {
                    Integer remove = bucket.remove(0);
                    arr[index++] = remove;
                }
            }

            // 开始下一个位数的比较
            radix *= 10;
        }
    }
```

# 时间复杂度
`O(n*k)`
假设n个参与排序的数, 且数的最大有K位, 那么
f(n) = k(k轮比较) * n(每轮操作n个数) = O(n*k)

# 空间复杂度

`O(n)`
多出一个链表结构存储排序的中间数值

# 完整代码

```java
class RadixSort {
    public static void sort(int[] arr) {

        if (arr == null || arr.length <= 1) {
            return;
        }

        // 初始化比较需要的`桶`空间
        ArrayList<ArrayList<Integer>> buckets = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            buckets.add(new ArrayList<>());
        }

        int radix = 1;
        // 循环比较各个位数
        while (true) {

            boolean isAllZero = true;

            // 按照位数放置到相应的桶位置
            for (int i = 0; i < arr.length; i++) {
                // 取出整数的个,十, 百, 千...
                int num = arr[i] / radix % 10;
                if (isAllZero && num != 0) {
                    isAllZero = false;
                }
                buckets.get(num).add(arr[i]);
            }

            if (isAllZero) {
                break;
            }

            // 回填Arr && 清空桶(buckets)中数据
            int index = 0;
            for (ArrayList<Integer> bucket : buckets) {
                while (!bucket.isEmpty()) {
                    Integer remove = bucket.remove(0);
                    arr[index++] = remove;
                }
            }

            // 开始下一个位数的比较
            radix *= 10;
        }
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

