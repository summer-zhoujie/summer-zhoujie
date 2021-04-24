---
layout:     post
title:      "十大排序算法(九), 桶排序"
subtitle:   ""
date:       2021-04-24 13:21:58
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 算法图示

**元素分配到桶中**

![Bucket_sort_1.svg.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6acbf1fd92054653a11e9bde797c824f~tplv-k3u1fbpfcp-watermark.image)

**对桶中的元素进行排序**

![Bucket_sort_2.svg.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/63ac4f6aa2f24974aebf23fb01266d66~tplv-k3u1fbpfcp-watermark.image)

1. 設置一個定量的陣列當作空桶子。
2. 尋訪序列，並且把項目一個一個放到對應的桶子去。
3. 對每個不是空的桶子進行排序。
4. 從不是空的桶子裡把項目再放回原來的序列中。

> 如果桶足够小就变成了`计数排序`, 计数排序是桶排序的一个特例

```java
public static void sort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }

        int max = 0;
        int min = 0;
        for (int i : arr) {
            max = Math.max(max, i);
            min = Math.min(min, i);
        }
        // 定义桶数量
        int BUCKET_SIZE = 10;
        int bucketCount = (max - min) / BUCKET_SIZE + 1;

        // 定义桶
        ArrayList<ArrayList<Integer>> buckets = new ArrayList<>();
        for (int i = 0; i < bucketCount; i++) {
            buckets.add(new ArrayList<Integer>());
        }

        // 初始化桶
        for (int i : arr) {
            int index = (i - min) / BUCKET_SIZE;
            buckets.get(index).add(i);
        }

        // 对各个桶数据进行排序
        for (ArrayList<Integer> bucket : buckets) {
            // 这里的排序算法决定着桶的算法复杂度
            Collections.sort(bucket);
        }

        // 桶拼接
        int p = 0;
        for (ArrayList<Integer> bucket : buckets) {
            for (Integer integer : bucket) {
                arr[p++] = integer;
            }
        }
    }
```

# 时间复杂度

假设桶的数量是K, 数组长度是n

f(n) = n(计算最大/小) + k(定义桶) + n(初始化桶) + n^2(对桶进行排序,可以是任一排序算法,1~n^2) + n(桶拼接)

f(n)max = n(3+n)+k = O(n^2)

f(n)min = 3n+k = O(n)

> 桶的时间复杂度取决于对每个桶进行排序时的算法时间效率, 最好的情况下是桶中只有一个数据不需要排序 O(1), 最差的排序算法复杂度是O(n^2)


# 空间复杂度

链表结构存储桶, 假设有n个数,空间复杂度为O(n)
> 需注意, 本文用ArrayList有个初始大小10, 空间复杂度大于 O(n)


# 完整代码

```java
class BucketSort {

    public static void sort(int[] arr) {
        if (arr == null || arr.length <= 1) {
            return;
        }

        int max = 0;
        int min = 0;
        for (int i : arr) {
            max = Math.max(max, i);
            min = Math.min(min, i);
        }
        // 定义桶数量
        int BUCKET_SIZE = 10;
        int bucketCount = (max - min) / BUCKET_SIZE + 1;

        // 定义桶
        ArrayList<ArrayList<Integer>> buckets = new ArrayList<>();
        for (int i = 0; i < bucketCount; i++) {
            buckets.add(new ArrayList<Integer>());
        }

        // 初始化桶
        for (int i : arr) {
            int index = (i - min) / BUCKET_SIZE;
            buckets.get(index).add(i);
        }

        // 对各个桶数据进行排序
        for (ArrayList<Integer> bucket : buckets) {
            // 这里的排序算法决定着桶的算法复杂度
            Collections.sort(bucket);
        }

        // 桶拼接
        int p = 0;
        for (ArrayList<Integer> bucket : buckets) {
            for (Integer integer : bucket) {
                arr[p++] = integer;
            }
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

# 参考
[维基百科-桶排序](https://zh.wikipedia.org/wiki/%E6%A1%B6%E6%8E%92%E5%BA%8F)

