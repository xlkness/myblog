---
title: 插入排序-《算法导论》学习笔记一
date: 2018-06-15 18:37:04
categories:
- 开发
tags:
- 数据结构/算法
---

算法导论第二章的第一小节是插入排序，也就是像打扑克牌整理扑克一样，从左边第二张开始，每张与前边排好序的扑克牌比较，比较到能插入的位置就插入，算法比较简单。

    #include <stdio.h>
    #include <unistd.h>
    #include <time.h>
    #include <string.h>
    #include <stdlib.h>

    void insert_sort0( int* arr, int idx )
    {
        int i = idx - 2;
        int tmp = arr[idx];

        arr[idx] = arr[idx - 1];

        for ( ; i >= 0; i-- ) {
            if ( arr[i] > tmp )
                arr[i + 1] = arr[i];
            else {
                arr[i + 1] = tmp;
                break;
            }
        }
        if ( i < 0 )
            arr[0] = tmp;
    }
    int insert_sort( int* arr, int len)
    {
        time_t t_old, t_new;
        t_old = time(NULL);
        if ( arr[0] > arr[1] ) {
            int tmp = arr[0];
            arr[0] = arr[1];
            arr[1] = tmp;
        }

        int i = 1;

        for ( ; i + 1< len; i++ ) {
            if ( arr[i + 1] < arr[i] ) {
                insert_sort0( arr, i + 1 );
            }
        }
        t_new = time(NULL);

        return (int)(t_new - t_old);
    }
    void sortLoop0( int *arr, int len )
    {
        insert_sort( arr, len );
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

我的ubuntu虚拟机环境是双核3代i5,2G内存，排序10-100000的随机10万的数组如下情况：

![insert_sort]

如果是100万的数组，耗时10几分钟；如果是1000万的数组，挂着跑了一晚上都没排好序。

对插入排序的一点思考：在写排序的时候想到当前数可以从排好序数组后往前比较，也可以从前往后比较，前一种比较一个不符合即可将比较的数组值后移一位，给待插入值让位，这样就在比较重顺便后移，如果是后一种比较插入方法，则是遍历排好序数组，直到找到符合的索引位置，将位置后边的数组后移，这样相当于每次都要遍历一遍排好序数组，因此不用此方法。

[insert_sort]:/img/insert_sort.jpeg "insert_jpeg"