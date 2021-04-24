---
layout:     post
title:      "十大排序算法(七), 归并排序"
subtitle:   ""
date:       2021-04-24 13:20:25
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 算法图解

![BA0D2CBA-2162-4BA2-A63E-E701D3C8E9C2.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/72e309c988034afbac2182a2ad2c3f91~tplv-k3u1fbpfcp-watermark.image)

典型的`分治法`, `分`是把排序问题分解到最小单位(即: 1个数排序), `治`把子树的排序结果向上合成上一层级父亲的排序结果, 下图描述的是`治`的过程

![0011886A-933B-4198-949B-2CEC66F10C27.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6c9ef2fc9c924ce3a2a14087ca738414~tplv-k3u1fbpfcp-watermark.image)

![945146A9-0DDD-4128-91F7-046736A9811D.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1189db0461374a1aa0f190141be54afa~tplv-k3u1fbpfcp-watermark.image)

1. 先递归对序列进行`分`解成最小单元
2. 逐级计算`治`的结果

# 实现

```java
private static void doSort(int[] arr, int[] tmp, int start, int end) {
        if (start < end) {
            int mid = (end + start) / 2;
            doSort(arr, tmp, start, mid);
            doSort(arr, tmp, mid + 1, end);
            merge(arr, tmp, start, mid, end);
        }
    }

private static void merge(int[] arr, int[] tmp, int start, int mid, int end) {
        int i = start;
        int j = mid + 1;
        int t = 0;
        while (i <= mid && j <= end) {
            if (arr[i] <= arr[j]) {
                tmp[t++] = arr[i++];
            }

            if (arr[i] > arr[j]) {
                tmp[t++] = arr[j++];
            }
        }

        while (i <= mid) {
            tmp[t++] = arr[i++];
        }

        while (j <= end) {
            tmp[t++] = arr[j++];
        }

        System.arraycopy(tmp, 0, arr, start, end - start + 1);
    }
```

# 时间复杂度

`O(nlogn)`
数组被分成了二叉树,二叉树的层高 log2n , 每层需要比对的次数 n
f(n) = n * log2n = O(nlogn)

# 空间复杂度

`O(n)`
用了一个temp数组,来缓存中间处理的数据, 当数据量很大的时候需要考虑这里的空间浪费问题

# 完整代码

```java
class MergeSort {

    private static void sort(int[] array) {
        if (array == null || array.length <= 1) {
            return;
        }

        int[] tmp = new int[array.length];
        int length = array.length;
        doSort(array, tmp, 0, length - 1);
    }

    private static void doSort(int[] arr, int[] tmp, int start, int end) {
        if (start < end) {
            int mid = (end + start) / 2;
            doSort(arr, tmp, start, mid);
            doSort(arr, tmp, mid + 1, end);
            merge(arr, tmp, start, mid, end);
        }
    }

    private static void merge(int[] arr, int[] tmp, int start, int mid, int end){
        int i = start;
        int j = mid + 1;
        int t = 0;
        while (i <= mid && j <= end) {
            if (arr[i] <= arr[j]) {
                tmp[t++] = arr[i++];
            }

            if (arr[i] > arr[j]) {
                tmp[t++] = arr[j++];
            }
        }

        while (i <= mid) {
            tmp[t++] = arr[i++];
        }

        while (j <= end) {
            tmp[t++] = arr[j++];
        }

        System.arraycopy(tmp, 0, arr, start, end - start + 1);
    }

    public static void main(String[] args) {
        int[] array = {111, 52, 77, 98, 36, 12, 12, 48};
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
[https://www.cnblogs.com/chengxiao/p/6194356.html](https://www.cnblogs.com/chengxiao/p/6194356.html)

