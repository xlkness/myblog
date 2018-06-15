---
title: 归并排序-《算法导论》学习笔记二
date: 2018-06-15 18:40:26
categories:
- 开发
tags:
- 数据结构/算法
---

算法导论第二章3小节讲到分治法，即将大问题化解为规模小一些的同类问题，这样先处理小问题，再合并两个小问题再解决，递归的思路。按照书上的伪代码，写了c算法。

    #include <stdio.h>
    #include <time.h>
    #include <stdlib.h>

    void merge( int *arr, int p,
        int q, int r )
    {
        int arrlen = r - p + 1;
        int *tmparr = (int *)malloc( sizeof(int) * arrlen );
        int i, j, k, tmp;

        for ( i = 0; i < arrlen; i++ )
            tmparr[i] = arr[i + p];

        j = 0;
        k = q - p + 1;

        for ( i = p; i < r + 1; i++ )
        {
            if ( tmparr[j] > tmparr[k] )
            {
                arr[i] = tmparr[k];
                k++;
            }
            else
            {
                arr[i] = tmparr[j];
                j++;
            }
            if ( j == q - p + 1 || k == r - p + 1 )
            {
                i++;
                break;
            }

        }
        for ( ; i < r + 1; i++ )
        {
            if ( j < q - p + 1 )
            {
                arr[i] = tmparr[j];
                j++;
                continue;
            }
            if ( k < r - p + 1 )
            {
                arr[i] = tmparr[k];
                k++;
                continue;
            }
        }

        free( tmparr );
        tmparr = NULL;
    }
    void mergeSort( int *arr, int p, int r )
    {
        int q;
        if ( p < r )
        {
            q = ( p + r ) / 2;
            mergeSort( arr, p, q );
            mergeSort( arr, q + 1, r );
            merge( arr, p, q, r );
        }
    }
    void sortLoop0( int *arr, int len )
    {
        mergeSort( arr, 0, len - 1 );
    }
    //排序前后做时间对比，精度sec
    void sortLoop( int *arr, int len )
    {
        time_t tt0 = time( NULL );
        printf("\tbefore sort:%s", ctime(&tt0));
        sortLoop0( arr, len );
        time_t tt1 = time( NULL );
        printf("\tafter sort:%s", ctime(&tt1));
        printf("\tsort the array cost %d(sec),%d(min)\n",
            (int)(tt1 - tt0), (int)((tt1 - tt0) / 60));
    }
    //初始化随机数组
    void initArr( int *arr, int lowV,
        int upV, int len )
    {
        srand( (int)time(NULL) );
        int i = 0;
        int size = upV - lowV;

        for ( ; i < len; i++ )
        {
            arr[i] = rand() % size + lowV;
        }
    }
    //打印数组
    void printArr( int *arr, int len )
    {
        int i = 0;
        for ( ; i < len; i++ )
            printf("%d ", arr[i]);
        printf("\n");
    }
    //这里写了个简单得数组检查，检查是否
    //是正确的递增序列(怕自己的代码没检查
    //边界条件偶尔有错误的排序数组)
    int check_arr_increase( int *arr, int len )
    {
        int i = 0;

        for ( ; i < len - 1; i++ )
        {
            if ( arr[i] <= arr[i + 1] )
                continue;
            else
            {
                printf("array isn't increase!!!\n");
                return -1;
            }

        }
        printf("array is increase!!!!\n");
        return 0;
    }
    int main( int argc, char **argv )
    {
        if ( argc != 4 )
        {
            printf("usage: ./execfile lowV upV len\n");
            return 0;
        }

        int lowV = atoi( argv[1] );
        int upV = atoi( argv[2] );
        unsigned int len = atoi( argv[3] );

        int *arr = NULL;
        arr = ( int *) malloc( len * sizeof(int) );

        initArr( arr, lowV, upV, len );
        //printArr( arr, len );
        sortLoop( arr, len );
        check_arr_increase( arr, len );
        //printArr( arr, len );

        free( arr );
        arr = NULL;

        return 0;
    }

分治法的算法复杂度可以计算，效率是很恐怖的，排序取值10-1000000值长度1000万的数组用时4s：

![merge_sort1]

排序取值10-1000000值长度1亿的数组用时85s：

![merge_sort2]

[merge_sort1]:/img/merge_sort1.jpeg ""
[merge_sort2]:/img/merge_sort2.jpeg ""