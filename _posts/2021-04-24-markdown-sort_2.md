---
layout:     post
title:      "十大排序算法(二), 快速排序"
subtitle:   ""
date:       2021-04-24 13:17:07
author:     "zhoujie"
catalog: true # 是否开启目录
tags:
    - 排序算法
---

# 算法图解
> `冒泡排序` 的优化

![217BC91A-9BAE-49AD-8BD0-132A9D400413.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0bfa9094831745eea29346aabb722110~tplv-k3u1fbpfcp-watermark.image)

`分治`的思想,
1. 随机找到第一个数字 6, 将比6小的挪到6左边, 比6大的挪到6右边, 分成两堆
2. 递归6的左边和右边
3. 重复1~2直到分解成最小单元(左右两边只有1个/0个元素)

如何实现 `将比6小的挪到6左边, 比6大的挪到6右边` , 参考 1.挖坑法 2.双指针法, 本文采用挖坑法实现
> [挖坑法 && 双指针法讲解](https://www.codenong.com/cs105997192/)

# 实现

```java
    private static void doSort(int[] arr, int start, int end) {

        if (start >= end) {
            return;
        }

        int pivot = arr[start];
        int pivotIndex = start;
        int left = start;
        int right = end;

        while (left < right) {


            // 坑在左边,往右边找
            while (left == pivotIndex && left < right) {
                if (arr[right] > pivot) {
                    right--;
                } else {
                    arr[pivotIndex] = arr[right];
                    pivotIndex = right;
                    break;
                }
            }

            // 坑在右边,往左边找
            while (right == pivotIndex && left < right) {
                if (arr[left] < pivot) {
                    left++;
                } else {
                    arr[pivotIndex] = arr[left];
                    pivotIndex = left;
                    break;
                }
            }
        }

        arr[left] = pivot;

        doSort(arr, start, left);
        doSort(arr, left + 1, end);
    }
```

# 算法时间复杂度

`O(n^2) ~ O(nlogn)`
取决于树的平衡性, 最差的n各节点的树的层次是n,最好的是log2n, 树的每层遍历比对的时间是n次

# 空间复杂度

`O(1)`

# 完整代码

```java
class QuickSort {

    private static void sort(int[] arr) {

        if (arr == null || arr.length <= 1) {
            return;
        }
        int start = 0;
        int end = arr.length - 1;
        doSort(arr, start, end);
    }

    private static void doSort(int[] arr, int start, int end) {

        if (start >= end) {
            return;
        }

        int pivot = arr[start];
        int pivotIndex = start;
        int left = start;
        int right = end;

        while (left < right) {


            // 坑在左边,往右边找
            while (left == pivotIndex && left < right) {
                if (arr[right] > pivot) {
                    right--;
                } else {
                    arr[pivotIndex] = arr[right];
                    pivotIndex = right;
                    break;
                }
            }

            // 坑在右边,往左边找
            while (right == pivotIndex && left < right) {
                if (arr[left] < pivot) {
                    left++;
                } else {
                    arr[pivotIndex] = arr[left];
                    pivotIndex = left;
                    break;
                }
            }
        }

        arr[left] = pivot;

        doSort(arr, start, left);
        doSort(arr, left + 1, end);
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
[https://www.sohu.com/a/246785807_684445/](https://www.sohu.com/a/246785807_684445/)

