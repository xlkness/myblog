---
title: 查找数组第i小的数-《算法导论》学习笔记十
date: 2018-06-19 20:32:13
categories:
- 开发
tags:
- 数据结构/算法
---

查找第i小的数利用了快速排序的一点思想，即以数组某个值作为比较值，然后遍历数组中除这个数以外的数，小于它的就放左边，大于它的放右边，然后作为比较值的数放中间，并返回比较值的下标，如果下标等于i，表示就找到了，如果下标大于i，又再次从大于此下标的数组递归进行此过程，同理，小于i就从小于此下标的数组递归进行此过程，代码：

    #include <stdio.h>
    #include <stdlib.h>

    int partition( int *arr, int p, int r )
    {
        int x = arr[r];
        int i = p - 1;
        int j = 0;
        int temp = 0;

        for ( j = p; j < r; j++ ) {
            if ( arr[j] <= x ) {
                i += 1;
                temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }

        temp = arr[i + 1];
        arr[i + 1] = arr[r];
        arr[r] = temp;

        return i + 1;
    }
    int randomized_partition( int *arr, int p, int r )
    {

        int i = rand() % (r - p) + p;
        int temp = arr[r];
        arr[r] = arr[i];
        arr[i] = temp;

        return partition( arr, p, r );
    }
    int randomized_select(
        int *arr,
        int p,
        int r,
        int i )
    {
        if ( p == r )
            return arr[p];

        int q = 0, k = 0;

        q = randomized_partition( arr, p, r );
        k = q - p + 1;
        if ( i == k )
            return arr[q];
        else if ( i < k )
            return randomized_select( arr, p, q - 1, i );
        else
            return randomized_select( arr, q + 1, r, i - k );
    }
    int main()
    {
        int arr[10] = {1,2,3,4,5,6,7,8,9,10};
        int len = 10;
        int i = 8;

        printf("第%d小的数是:%d\n", i, randomized_select(arr, 0, 9, i));

        return 0;
    }